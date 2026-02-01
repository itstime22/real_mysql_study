# 10.3 실행 계획 분석

> FORMAT 설정을 통해 JSON 또는 TREE 형태로 선택할 수도 있지만,
가독성 높은 테이블 포맷으로 실행 계획의 접근 방법과 최적화, 인덱스 사용 등을 이해해보자.
>

### 테이블 포맷 예시 출력

```sql
EXPLAIN
SELECT *
employees e
	INNER JOIN salaries s ON s.emp_no=e.emp_no
WHERE first_name='ABC';
```

| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | e | NULL | ref | PRIMARY, ix_firstname | ix_firstname | 58 | const | 1 | 100.00 | NULL |
| 1 | SIMPLE | s | NULL | ref | PRIMARY | PRIMARY | 4 | employees.e.emp_no | 10 | 100.00 | NULL |

### 레코드 접근 순서

- 각 라인(레코드)은 쿼리 문장에서 사용된 테이블 개수만큼 출력 (서브 쿼리로 생성된 임시 테이블 포함)
- 실행 순서는 위에서 아래로 순서대로 표시
- id 칼럼의 값이 작을수록 위쪽에, id 칼럼의 값이 클수록 아래쪽에 출력
- 출력된 실행 계획에서 위쪽에 출력된 결과일 수록 쿼리의 바깥(Outer) 부분이거나 먼저 접근한 테이블
- 아래쪽에 출력된 결과일수록 쿼리의 안쪽(Inner) 부분 또는 나중에 접근한 테이블에 해당

> 이제 실행 계획에 표시되는 각 칼럼이 어떤 것을 의미하는지, 어떤 값들이 출력될 수 있는지 알아보자.
>

## 10.3.1 id 칼럼

하나의 SELECT 문장은 다시 1개 이상의 하위(SUB) SELECT 문장을 포함할 수 있다.

```sql
SELECT ...
FROM (SELECT ... FROM tb_test1) tb1, tb_test2 tb2
WHERE tb1.id=tb2.id;
```

상단 쿼리 문장에 있는 각 SELECT는 아래 처럼 분리해서 생각해볼 수 있는데,

1. `SELECT … FROM tb_test1;`
2. `SELECT … FROM tb1, tb_test2 tb2 WHERE tb1.id=tb2.id;`

이렇게 SELECT 키워드 단위로 구분한 것을 “단위(SELECT) 쿼리”라고 표현하자.

실행 계획에서 가장 왼쪽에 표시되는 id 칼럼은 단위 SELECT 쿼리별로 부여되는 식별자 값으로,
위 예시 쿼리에서는 최소 2개의 id 값이 표시된다.

### 여러 개의 테이블이 조인되는 경우

```sql
EXPLAIN
SELECT e.emp_no, e.first_name, s.from_date, s.salary
FROM employees e, salaries s
WHERE e.emp_no=s.emp_no LIMIT 10;
```

| id | select_type | table | type | key | ref | rows | Extra |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | e | index | ix_firstname | NULL | 300252 | Using index |
| 1 | SIMPLE | s | ref | PRIMARY | employees.e.emp_no | 10 | NULL |

### 여러 개의 단위 SELECT 쿼리로 구성된 경우

> 쿼리 문장이 3개의 단위 SELECT로 구성되어 실행 계획 각 레코드가 각기 다른 id 값을 지닌다.
다만 실행 계획의 id 칼럼이 테이블의 접근 순서를 의미하지는 않는다는 것을 유념하자.
>

```sql
EXPLAIN
SELECT
( (SELECT COUNT(*) FROM employees) + (SELECT COUNT(*) FROM departments) ) AS total_count;
```

| id | select_type | table | type | key | ref | rows | Extra |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | PRIMARY | NULL | NULL | NULL | NULL | NULL | No table used |
| 3 | SUBQUERY | departments | index | ux_deptname | NULL | 9 | Using index |
| 2 | SUBQUERY | employees | index | ix_hiredate | NULL | 300252 | Using index |

## 10.3.2 select_type 칼럼

> **각 단위 SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 칼럼**
>

### 표시 가능 값

1. **SIMPLE**
    - UNION이나 서브쿼리를 사용하지 않는 단순한 SELECT 쿼리인 경우 표시되는 타입
    - 쿼리 문장이 아무리 복잡하더라도, 실행 계확에서 select_type이 SIMPLE인 단위 쿼리는 단 하나
    - 일반적으로 제일 바깥 SELECT 쿼리의 select_type이 SIMPLE로 표시된다.
    - **서브쿼리/UNION 없음 && 제일 바깥 쿼리: SIMPLE**
2. **PRIMARY**
    - UNION이나 서브쿼리를 가지는 SELECT  쿼리 실행 계획에서 가장 바깥쪽(Outer)에 있는 단위 쿼리
    - SIMPLE 처럼 PRIMARY 타입의 단위 SELECT 쿼리는 단 하나만 존재한다.
    - **서브쿼리/UNION 있음 && 제일 바깥 쿼리: PRIMARY**

> **바깥(Outer) 쿼리란?**
- 다른 SELECT에 의해 감싸지지 않은 SELECT
- 어떤 SELECT의 괄호 안에도 들어 있지 않은 최상위 SELECT
>
3. **UNION**
    - UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리
    - UNION의 첫 번째 단위 SELECT는 select_type이 **UNION**이 아닌 **임시 테이블(DERIVED)**로 표시

    ```sql
    EXPLAIN
    SELECT * FROM (
    	(SELECT emp_no FROM employees e1 LIMIT 10) UNION ALL
    	(SELECT emp_no FROM employees e2 LIMIT 10) UNION ALL
    	(SELECT emp_no FROM employees e3 LIMIT 10) ) tb;
    ```

    - 위 쿼리의 실행 계획에서 e1은 **DERIVED**로, e2&e3는 **UNION** 으로 select_type이 표기된다.
    - 세 개의 서브쿼리로 조회도니 결과르 UNION ALL로 결합해 임시 테이블을 만들어서 사용하기 때문
4. **DEPENDENT UNION**
    - UNION 타입 처럼 `UNION`이나 `UNION ALL`로 집합을 결합하는 쿼리에서 표시된다.
    - **DEPENDENT**는 `UNION`이나 `UNION ALL`로 결합된 단위 쿼리가 외부 쿼리에 의해 영향을 받는 것을 의미한다.

    ```sql
    EXPLAIN
    SELECT * 
    FROM employees e1 WHERE e1.emp_no IN (
    	SELECT e2.emp_no FROM employees e2 WHERE e2.first_name='Matt'
    	UNION
    	SELECT e3.emp_no FROM employees e3 WHERE e3.last_name='Matt'
    );
    ```

    - 위와 같은 쿼리에서 MySQL 옵티마이저는 IN 내부의 서브쿼리를 먼저 처리하지 않는다.
    - 외부의 employees 테이블을 먼저 읽은 후 서브쿼리를 실행하는데, 이때 employees 테이블의 칼럼 값이 서브쿼리에 영향을 준다.
    - 이렇게 내부 쿼리가 외부의 값을 참조해서 처리될 때 select_type에 DEPENDENT 키워드가 표시

| id | select_type | table | type | key | ref | rows | Extra |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | PRIMARY | e1 | ALL | NULL | NULL | 300252 | Using where |
| 3 | DEPENDENT SUBQUERY | e2 | eq_ref | PRIMARY | func | 1 | Using where |
| 2 | DEPENDENT UNION | e3 | eq_ref | PRIMARY | func | 1 | Using where |
| NULL | UNION RESULT | <union2,3> | ALL | NULL | NULL | NULL | Using temporary |
5. **UNION RESULT**
    - **UNION 결과를 담아두는 테이블을 의미**한다.
    - MySQL 8.0 부터 UNION ALL이 임시 테이블을 사용하지 않도록 기능이 개선됐다.
        - UNION, UNION DISTINCT는 여전히 임시 테이블에 결과를 버퍼링한다.
        - 즉 UNION ALL 사용 쿼리에서는 UNION RESULT 라인이 없어진다.
    - 실행 계획상에서 이 임시 테이블을 가리키는 라인의 select_type이 UNION RESULT이다.
        - 실제 쿼리에서 단위 쿼리가 아니기 때문에 별도의 id 값은 부여되지 않는다.
6. **SUBQUERY**
    - 일반적으로 서브쿼리는 여러 가지를 통틀어서 이야기할 때가 많아 헷갈릴 수 있다.
    - **select_type의 SUBQUERY는 FROM 절 이외에서 사용되는 서브쿼리만을 의미한다.**

<aside>
💡

**서브쿼리는 사용하는 위치에 따라 각각 다른 이름을 가진다.**

- **중첩된 쿼리(Nested Query)**: SELECT 되는 칼럼에 사용된 서브쿼리
- **서브쿼리(Subquery)**: WHERE 절에 사용된 경우 일반적으로 그냥 서브쿼리라고 한다.
- **파생 테이블(Derived Table)**: FROM 절에 사용된 서브쿼리를 MySQL에선 파생 테이블이라 하며, 일반적으로 RDBMS에서는 인라인 뷰(Inline View) 또는 서브 셀렉트(Sub Select)라 부른다.

**서브쿼리가 반환하는 값의 특성에 따라 다음과 같이 구분하기도 한다.**

- 스칼라 서브쿼리(Scalar Subquery): 하나의 값만(칼럼이 단 하나인 레코드 1건) 반환하는 쿼리
- 로우 서브쿼리(Low Subquery): 칼럼의 개수와 관계없이 하나의 레코드만 반환하는 쿼리
</aside>

7. **DEPENDENT SUBQUERY**
    - 서브쿼리가 바깥쪽(Outer) SELECT 쿼리에서 정의된 칼럼을 사용하는 경우 표시되는 select_type

    ```sql
    EXPLAIN
    SELECT e.first_name,
    	(SELECT COUNT(*)
    	FROM dept_emp de, dept_manager dm
    	WHERE dm.dept_no=de.dept_no AND de.emp_no=e.emp_no) AS cnt
    FROM employees e
    WHERE e.first_name='Matt'
    ```

    - 위와 같은 쿼리에서 안쪽의 서브쿼리 결과가 바깥쪽 SELECT 쿼리의 칼럼에 의존적이다.
    - 이 경우 DEPENDENT 키워드가 붙으며, DEPENDENT UNION 처럼 외부 쿼리가 먼저 수행된 후 내부 쿼리(서브쿼리)가 실행돼야 하므로 DEPENDENT 키워드가 없는 일반 서브쿼리보다 처리가 느리다.
8. **DERIVED**
    - **DERIVED는 단위 SELECT 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블을 생성하는 것을 의미**
        - MySQL 5.5 버전까지는 서브쿼리가 FROM 절에 사용된 경우 항상 select_type이 DERIVED인 실행 계획을 만들었다.
        - 5.6 버전 부터는 옵티마이저 옵션에 따라 FROM 절의 서브쿼리를 외부 쿼리와 통합하는 형태의 최적화가 수행되기도 한다.
    - **FROM 절의 서브 쿼리를 간단히 제거하고 조인으로 처리할 수 있는 경우 쿼리 최적화를 진행해주자.**
        - **쿼리 튜닝 시 실행 계획 확인 시 가장 먼저 select_type이 DERIVED 유뮤를 확인하자.**
        - **서브쿼리를 조인으로 풀어서 고쳐쓰는 습관을 들이면 어느 순간 조인으로 복잡한 쿼리 작성이 가능해진다.**
9. **DEPENDENT DERIVED**
    - FROM 절의 서브쿼리가 외부 쿼리를 참조할 경우의 select_type이다.
    - MySQL 8.0 이전까지는 FROM 절의 서브쿼리는 외부 칼럼을 사용할 수 없었다.
    - MySQL 8.0 부터 **래터럴 조인(LATERAL JOIN) 기능이 추가되면서 FROM 절의 서브쿼리에서도 외부 칼럼을 참조**할 수 있게 되었다.

    ```sql
    EXPLAIN
    SELECT employees e
    LEFT JOIN LATERAL
    	(SELECT *
    	FROM salaries s
    	WHERE s.emp_no=e.emp_no
    	ORDER BY s.from_date DESC LIMIT 2) AS s2 OM s2.emp_no=e.emp_no;
    ```

    - 위 쿼리는 래터럴 조인의 가장 태표적인 활용 예제로, employees 테이블 레코드 1건당
      salaries 테이블의 레코드를 최근 순서대로 최대 2건까지만 가져와서 조인을 실행한다.
    - 래터럴 조인 사용에는 LATERAL 키워드가 필수로, LATERAL 키워드가 없는 서브쿼리에서 외부 칼럼을 참조하면 오류가 발생한다.
10. **UNCACHEABLE SUBQUERY**
    - 하나의 쿼리 문장에 서브쿼리가 하나만 있더라도 그 서브쿼리가 한 번만 실행되는 것은 아니다.
    - 조건이 같은 서브쿼리는 재실행되지 않고 캐시 공간에서 꺼내와 사용한다.
    - 서브쿼리에 포함된 요소에 의해 캐시 자체가 불가능할 수 있는데, 이 경우 select_type이 UNCACHEABLE SUBQUERY로 표시된다.
    - 캐시를 사용하지 못하게 되는 요소로는 대표적으로 아래와 같은 경우가 있다.
        - 사용자 변수가 서브쿼리에 사용된 경우
        - NOT-DETERMINISTIC 속성의 스토어드 루틴이 서브쿼리 내에 사용된 경우
        - `UUID()`나 `RAND()` 같이 결과값이 호출될 때 마다 달라지는 함수가 서브쿼리에 사용된 경우
11. **UNCACHEABLE UNION**
    - 앞에서 설명한 **UNION과 UNCACHEABLE 키워드의 속성이 혼합**된 select_type을 의미한다.
12. **MATERIALZED**

   > 구체화란? 세미 조인에서 사용된 서브 쿼리를 통째로 임시 테이블로 생성한다는 의미
   >
    - MySQL 5.6 부터 도입된 select_type
    - 주로 FROM 절이나 IN(subquery) 형태의 쿼리에 사용된 서브쿼리의 최적화를 위해 사용된다.
    - MATERIALZED 최적화로 바깥 테이블과 임시 테이블의 조인 형태로 처리되었음을 확인할 수 있다.

    ```sql
    EXPLAIN
    SELECT * 
    FROM employees 
    WHERE e.emp_no IN (SELECT emp_no FROM salaries WHERE salary BETWEEN 100 AND 1000);
    ```

| id | select_type | table | type | key | rows | Extra |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | <subquery> | ALL | NULL | NULL | NULL |
| 1 | SIMPLE | e | eq_ref | PRIMARY | 1 | NULL |
| 2 | MATERIALIZED | salaries | range | ix_salary | 1 | Using where, Using index |

## 10.3.3 table 칼럼

> **실행 계획 각 레코드의 기준이 되는 테이블을 표시하는 칼럼**
>

MySQL 서버의 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시된다.

테이블에 별칭이 부여된 경우 별칭이 표시되며, 별도의 테이블을 사용하지 않는 경우 NULL로 표시된다.

table 칼럼에서 <derived N> 또는 <union M,N>과 같이 `<>` 로 둘러싸인 이름이 명시되는 경우가 많은데, 이 테이블은 임시 테이블을 의미한다.

`<>` 안에 항상 표사되는 숫자는 단위 SELECT 쿼리의 id 값을 지칭한다.

| id | select_type | table | type | key | rows | Extra |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | PRIMARY | <derived2> | ALL | NULL | 331143 | NULL |
| 1 | PRIMARY | e | eq_ref | PRIMARY | 1 | NULL |
| 2 | DERIVED | de | index | ix_empno_fromdate | 331143 | Using index |

위 실행 계획에서 <derived2> 는 단위 SELECT 쿼리의 id 값이 2인 실행 계획으로부터 만들어진 파생테이블을 가리킨다.

단위 SELECT 쿼리의 id 값이 2인 실행 계획은 dept_emp 테이블로 부터 SELECT된 결과가 저장된 파생 테이블이란 것을 일 수 있다.

위 3개의 칼럼을 통해 **실행 계획을 분석**한다면 아래와 같다.

1. 첫 번째 라인의 테이블이 <derived2>라는 것으로 보아 이 라인보다 id 값이 2인 라인이 먼저 실행되고 그 결과가 파생 테이블로 준비되어야 한다는 것을 알 수 있다.
2. 세 번째 라인(id 값이 2인 라인)을 보면 select_type 칼럼의 값이 DERIVED인데, 이를 통해 table 칼럼에 표시된 dept_emp 테이블을 읽어서 파생 테이블을 생성함을 알 수 있다.
3. 세 번째 라인의 분석이 끝났으므로 다시 실행 계획의 첫 번째라인으로 돌아가자
4. 첫 번째 라인과 두 번째 라인은 같은 id 값을 가지고 있는 것으로 봐서 2개 테이블(첫 번째 라인의 <derived2>와 두 번째 라인의 e 테이블)이 조인되는 쿼리라는 사실을 알 수 있다.
    - <derived2> 테이블이 e 테이블보다 먼저(윗 라인에) 표시되었기 때문에 <derived2> 테이블이 드라이빙 테이블이 되고, e 테이블이 드리븐 테이블이 된다는 것을 알 수 있다.
    - 즉 <derived2> 테이블을 먼저 읽어서 e 테이블로 조인을 실행했다는 것

**위 실행 계획에 해당되는 실제 쿼리는 아래와 같다.**

```sql
SELECT *
FROM
	(SELECT de.emp_no FROM dept_emp de GROUP BY de.emp_no) tb,
	employees e
WHERE e.emp_no=tb.emp_no;
```

## 10.3.4 partitions 칼럼

MySQL 8.0 버전 이후 EXPLAIN 명령으로 파티션 관련 실행 계획까지 확인 가능하게 변경되었다.

```sql
CREATE TABLE employees_2 (
	emp_no int NOT NULL,
	birth_date DATE NOT NULL,
	first_name VARCHAR(14) NOT NULL,
	last_name VARCHAR(16) NOT NULL,
	gender ENUM('M','F') NOT NULL,
	hire_date DATE NOT NULL,
	PRIMARY KEY (emp_no, hire_date)
) PARTITION BY RANGE COLUMNS(hire_date)
(PARTITION p1986_1990 VALUES LESS THAN ('1990-01-01'),
PARTITION p1991_1995 VALUES LESS THAN ('1996-01-01'),
PARTITION p1996_2000 VALUES LESS THAN ('2000-01-01'),
PARTITION p2001_2005 VALUES LESS THAN ('2006-01-01'));

INSERT INTO employees_2 SELECT * FROM employees;
```

위와 같이 범위를 통해 파티셔닝을 진행한 경우, 옵티마이저는 파티션의 범위 칼럼 조건을 통해 커리에서 필요한 파티션에 대해서만 쿼리를 수행하게된다.

이처럼 파티션이 여러 개인 테이블에서 불필요한 파티션을 빼고 쿼릴르 수행하기 위해 전급해야 할 것으로 찬단되는 테이블만 골라내는 과정을 파티션 프루닝(Partition pruning)이라고 한다.

| id | select_type | table | partitions | type | rows |
| --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | employees_2 | p1996_2000, p2001_2005 | ALL | 331143 |

위 실행 계확에서 type 칼럼이 ALL로 나타나는데, MySQL을 포함한 대부분의 RDBMS에서 지원하는 파티션은 물리적으로 개별 테이블처럼 별도의 저장 공간을 가지기에 풀 테이블 스캔을 진행하기 때문이다.

## 10.3.5 type 칼럼

> **MySQL 서버가 각 테이블의 레코드를 어떤 방식으로 읽었는지는 나타내는 칼럼**
>

일반적으로 쿼리를 튜닝할 때 인덱스를 효율적으로 사용하는지 확인하는 것이 중요하기에, 실행 계획에서 type 칼럼은 반드시 체크해야 할 중요한 정보이다.

MySQL에서는 type 칼럼을 “**조인 타입**”으로 소개하지만, 각 테이블의 **접근 방법**(Access type)으로 해석하자.

type 칼럼에 표시될 수 있는 값은 다음과 같다.

### 표시 가능 값

1. **system**
    - 레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법
    - InnoDB 테이블에선 나타나지 않고, MyISAM이나 MEMORY 테이블에서만 나타난다.
        - InnoDB 테이블에선 레코드가 하나라 할 지라도 ALL 또는 index로 표기될 것이다.
2. **const**
    - 테이블 레코드 건수와 관계없이, 쿼리가 PK나 Unique 키 칼럼을 이용하는 WHERE 조건절을 가지고 있으며, 반드시 1건을 반환하는 쿼리의 처리 방식을 const 라고 한다.
    - 다른 DBMS에서는 이를 유니크 인덱스 스캔(UNIQUE INDEX SCAN)이라고도 표현한다.
    - PK의 일부만 조건으로 사용할 때는 type 칼럼에 const 가 아닌 ref 로 표시된다.

<aside>
💡

**type 이름이 const 인 이유**

- type 칼럼이 const인 실행 계획은 MySQL의 옵티마이저가 쿼리를 최적화하는 단계에서 쿼리를 먼저 실행해서 통째로 상수화한다. 그래서 실행 계획의 type 칼럼 값이 const 인 것

```sql
EXPLAIN
SELECT COUNT(*)
FROM employees e1
WHERE first_name=(SELECT first_name FROM employees e1 WHERE emp_no=100001);
```

- 위 쿼리는 옵티마이저에서 최적화 되는 시점에 다음과 같은 쿼리로 변환된다.
- 즉 옵티마이저에 의해 상수화된 다음 쿼리 실행기로 전달되기 때문에 접근 방법이 const 인 것

```sql
SELECT COUNT(*)
FROM employees e1
WHERE first_name='Jasminko'; -- // Jasminko는 상단 서브쿼리의 결과값
```

</aside>

3. **eq_ref**
    - 조인에서 처음 읽은 테이블의 칼럼값을, 그 다음 읽어야 할 테이블의 PK나 유니크 키 칼럼의 검색 조건에 사용할 때를 가리켜 eq_ref 라고 한다.
    - 이때 두 번째 이후에 읽는 테이블의 type 칼럼에 eq_ref가 표시되며, 다중 칼럼으로 만들어진 PK나 유니크 인덱스라면 인덱스이 모든 칼럼이 비교 조건에 사용돼야만 eq_ref 접근 방법이 사용될 수 있다.
        - 즉, 조인에서 두 번째 이후에 읽는 테이블에서 반드시 1건만 존재한다는 보장이 있어야 사용 가능

    ```sql
    EXPLAIN
    SELECT * FROM dept_emp de, employees e
    WHERE e.emp_no=de.emp_no AND de.dept_no='d005';
    ```

   위 예시 쿼리를 분석해보자

    - 첫 번째 라인과 두 번째 라인의 id 값이 1로 같으므로 두 개의 테이블이 조인으로 실행됨을 알 수 있다.
    - dept_emp 테이블이 상단에 있기에, 먼저 읽고 동등 조건으로 employees 테이블을 검색한다.
    - employees 테이블의 emp_no는 PK기에 실행 계획의 2번째 라인은 type 칼럼이 eq_ref로 표기된다.

| id | select_type | table | type | key | key_len | rows |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | de | ref | PRIMARY | 16 | 165571 |
| 1 | SIMPLE | e | eq_ref | PRIMARY | 4 | 1 |
4. **ref**
    - eq_ref와 달리 조인의 순서와 관계없이 사용되며, PK나 유니크 키의 제약 조건도 없다.
    - 인덱스의 종류와 관계없이 동등(Equal) 조건으로 검색할 때는 ref 접근 방법이 사용된다.
    - 반환되는 레코드가 반드시 1건이라는 보장은 없기에 const나 eq_ref 보단 느리다.
    - 하지만 동등한 조건으로만 비교되므로 매우 빠른 레코드 조회 방법의 하나다.

<aside>
💡

**동등 연산자인 경우의 type 칼럼 (`=` 또는 `<=>`)**

| **const**: 조인의 순서와 관계없이 PK나 유니크 키의 모든 칼럼에 대해 동등(Equal) 조건으로 검색(반드시 1건의 레코드만 반환)
| **eq_ref**: 조인에서 첫 번째 읽은 테이블의 칼럼값을 이용해 두 번째 테이블을 PK나 유니크 키로 동등 조건 검색(두 번째 테이블은 반드시 1건의 레코드만 반환)
| **ref**: 조인의 순서와 인덱스 종류에 관계없이 동등 조건으로 검색
</aside>

5. **fulltext**
    - MySQL 서버의 **전문 검색(Full-text Search) 인덱스를 사용해 레코드를 읽는 접근 방법**을 의미
    - MySQL 서버에서 전문 검색 조건의 우선순위는 상당히 높은 편이며, 쿼리에서 전문 인덱스를 사용하는 조건과 일반 인덱스가 공존할 경우 const/eq_ref/ref가 아니면 전문 인덱스가 선택되어 사용된다.
6. **ref_or_null**
    - **ref 접근 방법과 동일하지만, NULL 비교가 추가된 형태**이다.
    - 실제 업무에서는 많이 활용되지 않지만, 사용된다면 나쁘지 않은 접근 방법 정도로 기억해두자.
7. **unique_subquery**
    - WHERE 조건절에서 사용될 수 있는 IN(subquery) 형태의 쿼리를 위한 접근 방법
    - unique_subquery 라는 이름대로 **서브쿼리에서 중복되지 않는 유니크한 값만 반환할 때 사용**한다.
8. **index_subquery**
    - IN 연산자 특성상 괄호 안에 있는 값 목록에서 중복된 값을 제거해야한다.
    - **서브쿼리 결과의 중복된 값을 인덱스를 이용해서 제거할 수 있을 때 사용되는 접근 방법**이다.
9. **range**
    - 우리가 익히 알고있는 **인덱스 레인지 스캔 형태의 접근 방법**
    - 인덱스를 하나의 값이 아닌 범위로 검색하는 경우를 의미하는데, 주로 `<`, `>`, `IS NULL`, `BETWEEN`, `IN`, `LIKE` 등의 연산자를 이용해 인덱스를 검색할 때 사용된다.
    - 일반적으로 애플리케이션의 쿼리가 가장 많이 사용하는 접근 방법인데, 소개된 type 들 중에서 상당히 우선순위가 낮음을 알 수 있지만, range 접근 방법도 상당히 빠르며 최적의 성능이 보장된다.
10. **index_merge**
    - 2개 이상의 인덱스를 이용해 각각의 검색 결과를 만들어낸 후, 그 결과를 병합해서 처리하는 방식
    - 여러 인덱스를 읽어야 하기에 일반적으로 range 접근 방법보다 효율성이 떨어진다.
    - 전문 검색 인덱스 사용 쿼리에서는 index_merge가 적용되지 않는다.
    - Index_merge 접근 방법으로 처리된 결과는 항상 2개 이상의 집합이 되기 때문에 그 두 집합의 교집합이나 합집합, 또는 중복 제거와 같은 부가적인 작업이 더 필요하다.

    ```sql
    EXPLAIN
    SELECT * FROM employees
    WHERE emp_no BETWEEN 10001 AND 11000
    	OR first_name='Smith';
    ```

| id | type | key | key_len | Extra |
| --- | --- | --- | --- | --- |
| 1 | index_merge | PRIMARY, ix_firstname | 4,58 | Using union(PRIMARY, ix_firstname); Using where |
    - 위 쿼리는 두 개의 조건이 OR 연산자로 연결된 쿼리이다.
        - OR로 연결된 두 개 조건이 모두 각각 다른 인덱스를 최적으로 사용할 수 있는 조건이다.
        - 그래서 `BETWEEN 10001 AND 11000` 조건은 employees 테이블의 PK를 통해 조회한다.
        - `first_name='Smith'` 조건은 ix_firstname 인덱스를 이용해 조회한다.
        - 이후 두 결과를 병합하는 형태로 처리하는 실행 계획을 만들어 낸다.
11. **index**
    - index 접근 방법은 많은 사람이 자주 오해하는 접근 방식으로, 이름만 보고 효율적으로 인덱스를 사용한다고 생각할 수 있다.
    - 하지만 **index 접근 방법은 인덱스를 처음부터 끝까지 읽는 인덱스 풀 스캔을 의미**한다.
        - range 접근 방법 처럼 효율적으로 인덱스의 필요한 부분만 읽는 방식이 아니다.
    - **풀 테이블 스캔 방식과 비교했을 때 읽는 레코드 건수는 같지만, 일반적으로 데이터 파일 전체보다 크기가 작으므로 인덱스 풀 스캔이 테이블 풀 스캔보다 빠르게 처리되며, 쿼리의 내용에 따라 정렬된 인덱스의 장점을 이용할 수 있기에 훨씬 효율적이라 할 수 있다.**
    - index 접근 방법은 다음 조건 중 1&2 조건을 충족하거나 1&3 조건을 충족하는 쿼리에서 사용된다.
        1. range 나 const, ref 같은 접근 방법으로 인덱스를 사용하지 못하는 경우
        2. 인덱스에 포함된 칼럼만으로 처리할 수 있는 쿼리인 경우(데이터 파일을 읽지 않아도 될 때)
        3. 인덱스를 이용해 정렬이나 그루핑 작업이 가능한 경우(별도의 정렬 작업을 피할 수 있을 때)
    - `SELECT * FROM departments ORDER BY dept_name DESC LIMIT 10;`
        - 위 쿼리는 인덱스를 처음부터 끝까지 읽는 index 접근 방법이지만, LIMIT 조건으로 인해 상당히 효율적이다.
12. **ALL**
    - 우리가 흔히 알고 있는 **풀 테이블 스캔을 의미하는 접근 방법**이다.
    - 테이블을 처음부터 끝까지 전부 읽어서 불필요한 레코드를 제거하고 반환한다.
    - 지금까지의 접근 방법으로는 처리할 수 없을 때 가장 마지막에 선택하는 가장 비효율적인 방법이다.
    - 다른 DBMS와 같이 InnoDB도 풀 테이블 스캔이나 인덱스 풀 스캔과 같은 대량의 디스크 I/O를 유발하는 작업을 위해 한꺼번에 많은 페이지를 읽어들이는 기능을 제공한다. - 리드 어헤드(Read Ahead)
    - **위 이유로 데이터 웨어하우스나 배피 프로그램처럼 대용량의 레코드를 처리하는 쿼리에서는 잘못 튜닝된 쿼리(억지로 인덱스를 사용하게 튜닝된 쿼리)보다 더 나은 접근 방법이기도 하다.**
    - **쿼리를 튜닝한다는 것이 무조건 인덱스 풀 스캔이나 테이블 풀 스캔을 사용하지 못하게 하는 것은 아니라는 점을 기억하자.**

## 10.3.6 possible_keys 칼럼

> 옵티마이저가 최적의 실행 계획을 만들기 위해 후보로 선정했던 접근 방법에서 사용되는 인덱스의 목록
>

**말 그대로 사용될법했던 인덱스의 목록**이며, 실행 계획을 보면 그 테이블의 모든 인덱스가 목록에 포함되어 나오는 경우가 허다하기에 쿼리 튜닝에 큰 도움이 되지는 않는다.

possible_keys 칼럼에 나열된 인덱스 이름만 보고 해당 인덱스를 사용한다고 판단하지 말자.

## 10.3.7 key 칼럼

> **최종 선택된 실행 계획에서 사용하는 인덱스**
>

쿼리 튜닝 시 key 칼럼에 의도했던 인덱스가 표시되는지 확인하는 것이 중요하다.

표시되는 값이 PRIMARY 인 경우에는 PK를 사용한다는 의미이며, 그 이외의 값은 모두 테이블이나 인덱스 생성 시 부여했던 고유 이름이다.

실행 계획의 type 칼럼이 index__merge가 아닌 경우에는 반드시 테이블 하나당 하나의 인덱스만 이용할 수 있다.

실행 계획의 type이 ALL 일 때 처럼 인덱스를 전혀 사용하지 못하는 경우 key 칼럼은 NULL로 표시된다.

index_merge 실행 계획이 사용될 때는 2개 이상의 인덱스가 사용되는데, 이때는 key 칼럼에 여러 개의 인덱스가 `.` 로 구분되어 표시된다.

| id | type | key | key_len | Extra |
| --- | --- | --- | --- | --- |
| 1 | ALL | PRIMARY,ix_firstname | 4,58 | Using union(PRIMARY,ix_firstname); Using where |

## 10.3.8 key_len 칼럼

> **쿼리를 처리하기 위해 다중 칼럼으로 구성된 인덱스에서 몇 개의 칼럼까지 사용했는지 알려주는 칼럼
정확하게는 인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려주는 값**
>

실제 업무 사용 테이블에선 단일 칼럼으로만 만들어진 인덱스보다 다중 칼럼으로 만들어진 인덱스가 더 많다.

```sql
EXPLAIN
SELECT * FROM dept_emp WHERE dept_no='d005';
```

| id | select_type | table | type | key | key_len |
| --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | dept_emp | ref | PRIMARY | 16 |

위 예제는 두 개의 칼럼(dept_emp)으로 구성된 PK를 가지는 dept_emp 테이블을 조회하는 쿼리이다.

위 쿼리에서는 dept_emp의 PK 중에서 dept_no 만 비교에 사용하기에, key_len 값이 16으로 표시된다.

dept_no 칼럼의 타입이 CHAR(4)이기 때문에 PK에서 앞쪽 16바이트만 유효하게 사용했다는 의미이다.

NULLABLE 칼럼으로 정의된 경우 NULL 여부 저장을 위한 1바이트가 추가되어 표시된다.

## 10.3.9 ref 칼럼

> 참조 조건(Equal 비교 조건)으로 어떤 값이 제공됐는지 보여주는 칼럼
>

상숫값을 지정했다면 ref 칼럼의 값은 const로 표시되고, 다른 테이블의 칼럼값이면 그 테이블명과 칼럼명이 표시된다.

이 칼럼에 출력되는 내용은 크게 신경쓰지 않아도 무방하나, 아래와 같은 케이스는 주의해서 봐보자.

쿼리의 실행 계획에서 ref 칼럼의 값이 func라고 표시되는 경우, Function의 줄임말로 참조용으로 사용되는 값을 그대로 사용한 것이 아니라, 콜레이션 변환이나 값 자체의 연산을 거쳐 참조되었다는 것을 의미한다.

```sql
EXPLAIN
SELECT * 
FROM employees e, dept_emp de WHERE e.emp_no=(de.emp_no-1);
```

| id | select_type | table | type | key | ref |
| --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | de | ALL | NULL | NULL |
| 1 | SIMPLE | e | eq_ref | PRIMARY | func |

## 10.3.10 rows 칼럼

> 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수를 보여준다.
>

이 값은 각 스토리지 엔진별로 가지고 있는 통계 정보를 참조해 MySQL 옵티마이저가 산출해 낸 예상값이기에, 정확하지는 않다.

또한 rows 칼럼에 표시되는 값은 반환하는 레코드의 예측치가 아니라 쿼리를 처리하기 위해 얼마나 많은 레코드를 읽고 체크해야 하는지를 의미한다.

실행 계획의 rows 칼럼에 출력되는 값과 실제 쿼리 결과로 반환된 레코드 건수는 일치하지 않는 경우가 많다.

## 10.3.11 filtered 칼럼

> `rows` → 스토리지 엔진에서 **읽을 것으로 예상되는 행 수**
`filtered` → 읽을 것으로 예상되는 행 중에서, **조건 적용 후 남을 것(통과)으로 예상되는 레코드의 비율**
>

옵티마이저는 각 테이블에서 일치하는 레코드 개수를 가능하면 정확히 파악해야 좀 더 효율적인 실행 계획을 수립할 수 있다. 실행 계획에서 **rows 칼럼의 값은 인덱스를 사용하는 조건에만 일치하는 레코드 건수**를 예측한 것이다.

대부분의 쿼리에서 WHER 절에 사용되는 조건이 모든 인덱스를 사용할 수 있는 것은 아니며, 특히 조인이 사용되는 경우에는 WHERE 절에서 인덱스를 사용할 수 있는 조건도 중요하지만 인덱스를 사용하지 못하는 조건에 일치하는 레코드 건수를 파악하는 것도 매우 중요하다.

```sql
EXPLAIN
SELECT *
FROM employees e, salaries s
WHERE e.firstname='Matt'
	AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
	AND s.emp_no=e.emp_no
	AND s.from_date BETWEEN '1990-01-01' AND '1991-01-01'
	AND s.salary BETWEEN 50000 AND 60000;
```

| id | select_type | table | type | key | rows | filtered |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | e | ref | ix_firstname | 233 | 16.03 |
| 1 | SIMPLE | s | ref | PRIMARY | 10 | 0.48 |

위 실행 계획에서 employees 테이블에서 salaries 테이블로 조인을 수행한 레코드 건수는 대략
37(233 * 0.1603)건 정도 였다는 것을 알 수 있다.

**조인의 선행 테이블을 바꿔 순서를 바꾼다면 어떨까?**

```sql
EXPLAIN
SELECT /*+ JOIN_ORDER(s, e) *
FROM employees e, salaries s
WHERE e.firstname='Matt'
	AND e.hire_date BETWEEN '1990-01-01' AND '1991-01-01'
	AND s.emp_no=e.emp_no
	AND s.from_date BETWEEN '1990-01-01' AND '1991-01-01'
	AND s.salary BETWEEN 50000 AND 60000;
```

| id | select_type | table | type | key | rows | filtered |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | SIMPLE | s | range | ix_salary | 3314 | 11.11 |
| 1 | SIMPLE | e | eq_ref | PRIMARY | 1 | 5.00 |

이 경우 대략 368(3314 * 0.1111)건이 조건에 일치해서 employees 테이블로 조인을 수행했을 것이다.

**최종적으로 일치하는 레코드 건수가 적은 테이블이 드라이빙 테이블로 선정될 가능성이 높기 때문이다.**