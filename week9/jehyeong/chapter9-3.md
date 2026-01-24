# 9.3.2 조인 최적화 알고리즘

MySQL에는 5.0 버전부터 있던 조인 쿼리의 실행 계획 최적화를 위한 알고리즘이 2개 있다.

테이블이 많아질 수록 최적화된 실행 계획 탐색이 어려워지고,
하나의 쿼리에서 조인되는 테이블의 개수가 많아지면 실행 꼐획을 수립하는 데만 몇 분이 걸릴 수도 있다.

간단히 4개의 테이블을 조인하는 쿼리 문장이 조인 옵티마이저 알고리즘에 따라 어떻게 처리되는지 살펴보자.

```sql
SELECT *
FROM t1, t2, t3, t4
WHERE ...
```

## 9.3.2.1 Exhaustive 검색 알고리즘

MySQL 5.0과 그 이전 버전에서 사용되던 조인 최적화 기법

FROM 절에 명시된 모든 테이블의 조합에 대해 실행 계획의 비용을 계산해서 최적의 조합 1개를 찾는 방법

예를들어 테이블이 20개라면 가능한 조인 조합은 20! == 3628800 개가 되어버린다.

이전 버전의 Exhaustive 검색 알고리즘에선 테이블이 10개만 넘어도 실행계획 수립에 몇 분이 걸린다.

테이블이 10개에서 1개만 더 늘어나도 11배의 시간이 더 걸린다.

## 9.3.2.2 Greedy 검색 알고리즘

- `optimizer_search_depth`가  1~62면 Greedy 검색 대상을 지정 개수로 한정해 실행 계획을 산출한다.
- `optimizer_search_depth`가 0이면 옵티마이저가 자동으로 최적 조인 테이블 개수를 정한다.
- `optimizer_search_depth`가 0이면 성능에 영향을 미칠 수 있으니 4~5 정도로 설정하자.

1. 전체 N개 테이블중에서 `optimizer_search_depth` 시스템 설정 변수에 정의된 개수의 테이블로 가능한 조인 조합 생성
2. 1번에서 생성된 조인 조합 중에서 최소 비용의 실행 계획 하나를 선정
3. 2번에서 선정된 실행 계획의 첫 번째 테이블을 “부분 실행 계획”의 첫 번째 테이블로 선정
4. 전체 N-1 개의 테이블 중(3번에서 선택된 테이블 제외) `optimizer_search_depth` 시스템 설정 변수에 정의된 개수의 테이블로 가능한 조인 조합을 생성
5. 4번에서 생성된 조인 조합들을 하나씩 3번에서 생성된 “부분 실행 계획”에 대입해 실행 비용을 계산
6. 5번의 비용 계산 결과, 최적의 실행 계획에서  두 번째 테이블을 3번에서 생성된 “부분 실행 계확”의 두 번째 테이블로 선정
7. 남은 테이블이 모두 없어질 때 까지 4~6번 까지의 과정을 반복 실행하면서 “부분 실행 계획”에 테이블의 조인 순서를 기록
8. 최종적으로 “부분 실행 계획”이 테이블의 조인 순서로 결정됨

> `optimizer_prune_level`이 1이면 옵티마이저는 조인 순서 최적화에 경험 기반의 휴리스틱 알고리즘을 사용한다. 0이면 휴리스틱 최적화가 적용되지 않기에, 특별한 요건이 없다면 1로 설정하자. (Default)
>

# 9.4 쿼리 힌트

MySQL 서버는 우리가 서비스하는 비즈니스를 100% 이해하지는 못하기에, 옵티마이저에게 쿼리의 실행 계획을 어떻게 수립해야할 지 알려줄 수 있는 방법이 필요하다. → 이 방법이 바로 힌트

MySQL 서버에서 사용 가능한 쿼리 힌트는 `인덱스 힌트` / `옵티마이저 힌트` 2가지로 구분할 수 있다.

## 9.4.1 인덱스 힌트

`STRAIGHT_JOIN` 과 `USE INDEX` 를 포함한 인덱스 힌트들은 사용 시  모두 SQL 문법에 맞게 사용해야 하기에 ANSI-SQL 표준 문법을 준수하지 못하게 된다.

가능하다면 MySQL 5.6 버전부터 추가된 옵티마이저 힌트를 사용하자. MySQL 서버를 제외한 다른 RDBMS에서는 주석으로 해석하기에 ANSI-SQL 표준을 준수한다고 볼 수 있다.

### 9.4.1.1 STRAIGHT_JOIN

SELECT, UPDATE, DELETE 쿼리에서 여러 개의 테이블이 조인되는 경우 조인 순서를 고정하는 역할

```sql
SELECT STRAIGHT_JOIN // 또는 SELECT /*! STRAIGHT_JOIN */
	e.first_name, e.last_name, d.dept_name
FROM employees e, dept_emp de, departsments d
WHERE e.emp_no=de.emp_no
	AND d.dept_no=de.dept_no;
```

FROM 절에 명시된 테이블의 순서대로 (e→de→d) 조인을 수행한다.

`STRAIGHT_JOIN` 힌트와 비슷한 역할을 하는 옵티마이저 힌트로는 다음과 같은 것들이 있다.

- `JOIN_FIXED_ORDER` → 동일한 역할
- `JOIN_ORDER` / `JOIN_PREFIX` / `JOIN_SUFFIX` → 일부 테이블의 조인 순서에 대해서만 제안하는 힌트

### 9.4.1.2 USE INDEX / FORCE INDEX / IGNORE INDEX

사용하려는 인덱스를 가지는 테이블 뒤에 힌트를 명시해야한다.

MySQL 옵티마이저는 사용할 인덱스를 무난하게 잘 선택하지만, 3~4개 이상의 칼럼을 포함하는 비슷한 인덱스가 여러 개존재하는 경우 가끔 실수를 한다.

이때 강제로 특정 인덱스를 사용하도록 힌트를 추가한다.

**인덱스 힌트 종류**

- **USE INDEX**
   - 가장 자주 사용되는 인덱스 힌트로, MySQL 옵티마이저에게 특정 테이블의 인덱스를 사용하도록 권장하는 힌트 → 대부분 채택하지만, 반드시 사용하는 것은 아니다.
- **FORCE INDEX**
   - 기능 자체는 USE INDEX와 동일하지만, 옵티마이저에 미치는 영향이 더 크다.
- **IGNORE INDEX**
   - 특정 인덱스를 사용하지 못하게 하는 용도로 사용하는 힌트
   - 옵티마이저가 풀 테이블 스캔을 사용하도록 유도하기위해 사용하기도 한다.

**인덱스 힌트 용도 명시**

- **USE INDEX FOR JOIN**
   - 테이블 간 조인뿐만 아니라 레코드를 검색하기 위한 용도까지 포함하는 용어
   - MySQL 에서는 하나의 테이블로 부터 데이터를 검색하는 작업도 JOIN이라고 표현하기 때문
- **USE INDEX FOR ORDER BY**
   - 명시된 인덱스를 ORDER BY 용도로만 사용할 수 있게 제한
- **USE INDEX FOR GROUP BY**
   - 명시된 인덱스를 GROUP BY 용도로만 사용할 수 있게 제한

### 9.4.1.3 SQL_CALC_FOUND_ROWS

MySQL의 LIMIT을 사용하는 경우 레코드가 LIMIT에 명시된 수만큼 만족 레코드를 찾으면 즉시 검색을 멈춤

`SQL_CALC_FOUND_ROWS` 힌트가 포함된 쿼리는 LIMIT을 만족하는 수만큼 찾아도 끝까지 검색을 수행한다.

- 이 경우 `FOUND_ROWS()` 라는 함수를 이용해 LIMIT을 제외한 조건을 만족하는 레코드가 전체 몇 건 이었는지를 알아낼 수 있다.

## 9.4.2 옵티마이저 힌트

### 9.4.2.1 옵티마이저 힌트 종류

> 옵티마이저 힌트는 영향 범위에 따라 다음 4개 그룹으로 나누어 볼 수 있다.
이 구분으로 인해 힌트의 사용 위치가 달라지는 것은 아니다.
>
- **인덱스**: 특정 인덱스의 이름을 사용할 수 있는 옵티마이저 힌트
- **테이블**: 특정 테이블의 이름을 사용할 수 있는 옵티마이저 힌트
- **쿼리 블록**: 특정 쿼리 블록에 사용할 수 있는 옵티마이저 힌트로서, 이름을 명시하는 것이 아니라 힌트가 명시된 쿼리 블록에 대해서만 영향을 미치는 옵티마이저 힌트
- **글로벌(쿼리 전체)**: 전체 쿼리에 대해서 영향을 미치는 힌트

```sql
-- // 사용 예시
EXPLAIN
SELECT /*+ INDEX(employees ix_firstname) */ *
FROM employees
WHERE firstname='Matt';
```

### 9.4.2.2 MAX_EXECUTION_TIME

> 옵티마이저 힌트 중에서 유일하게 쿼리의 실행 계획에 영향을 미치지 않는 힌트
>

단순히 쿼리의 최대 실행 시간을 설정하는 힌트다.

밀리초 단위의 시간을 설정하며, 쿼리가 지정된 시간을 초과하면 다음과 같이 쿼리가 실패한다.

```sql
-- // 사용 예시
SELECT /*+ MAX_EXECUTION_TIME(100) */ *
FROM employees
WHERE firstname='Matt';

ERROR 3024 (HY000): Query execution was interrupted, maximum statement execution time exceeded
```

### 9.4.2.3 SET_VAR

> 특정 쿼리에서 `optimizer_switch` 시스템 변수를 제어하기 위해 사용하는 옵티마이저 힌트
>

실행 계획을 바꿀 뿐만 아니라 조인 버퍼나 정렬용 버퍼(소트 버퍼)의 크기를 일시적으로 증가시켜 대용량 처리 쿼리의 성능을 향상시키는 용도로 사용할 수 있다.

```sql
-- // 사용 예시
EXPLAIN
SELECT /*+ SET_VAR(optimizer_switch='index_merge_intersection=off') */ *
FROM employees
WHERE firstname='Matt';
```

### 9.4.2.4 SEMIJOIN & NO_SEMIJOIN

> 세미 조인의 어떤 세부 전략을 사용할지 제어하기위해 사용한다.
>

| **최적화 전략** | **힌트** |
| --- | --- |
| **Duplicate Weed-out** | `SEMIJOIN(DUPSWEEDOUT)` |
| **First Match** | `SEMIJOIN(FIRSTMATCH)` |
| **Loose Scan** | `SEMIJOIN(LOOSESCAN)` |
| **Materialization** | `SEMIJOIN(MATERIALIZATION)` |
| **Table Pull-out** | **없음** (사용할 수 있다면 항상 더 나은 성능을 보장하기에 따로 힌트 설정 X) |

### 9.4.2.5 SUBQUERY

> 세미 조인 최적화가 사용되지 못할 때 사용하는 최적화 방법으로, 다음 2가지 형태로 최적화할 수 있다.
주로 안티 세미 조인 최적화에 사용되는 경우가 많다.
>

| **최적화 방법** | **힌트** |
| --- | --- |
| **IN-to-EXISTS** | `SUBQUERY(INTOEXISTS)` |
| **Materialization** | `SEMIJOIN(MATERIALIZATION)` |

### 9.4.2.6 BNL & NO_BNL & HASHJOIN & NO_HASHJOIN

BNL은 블록 네스티드 루프 조인을 위해 사용하는 옵티마이저 힌트였다.

MySQL 8.0.20 버전부터 해시 조인 알고리즘이 블록 네스티드 루프 조인을 대체하였는데,
**BNL 힌트도 MySQL 8.0.20 버전 이후 해시 조인 사용을 유도하는 힌트로 용도가 변경되었다.**

**HASHJOIN & NO_HASHJOIN 힌트는 MySQL 8.0.18 버전에서만 유효하며 이후 버전에는 효력이 없다.**

### 9.4.2.7 JOIN_FIXED_ORDER & JOIN_ORDER & JOIN_PREFIX & JOIN_SUFFIX

> 인덱스 힌트인 `STRAIGHT_JOIN` 은 FROM 절에 사용된 테이블의 순서대로 조인 실행을 유도하는 힌트
>
- **`JOIN_FIXED_ORDER` → FROM 절의 테이블 순서대로 조인을 실행하게 하는 힌트**
- **`JOIN_ORDER` → FROM 절이 아닌 힌트에 명시된 테이블의 순서대로 조인을 실행하는 힌트**
- **`JOIN_PREFIX` → 조인에서 드라이빙 테이블만 강제하는 힌트**
- **`JOIN_SUFFIX` → 조인에서 드리븐 테이블(마지막 조인 대상)만 강제하는 힌트**

```sql
-- // 사용 예시
SELECT /*+ JOIN_ORDER(d, de) */ *
FROM employees e
	INNER_JOIN dept_emp de ON de.emp_no=e.emp_no
	INNER_JOIN departmemts d ON d.dept_no=e.dept_no;
```

### 9.4.2.8 MERGE & NO_MERGE

예전 버전의 MySQL 서버에서는 FROM 절에 사용된 서브쿼리를 항상 내부 임시 테이블로 생성했다.

**이렇게 생성된 임시 테이블이 파생 테이블(Derived Table)인데, 불필요한 자원 소모를 유발하여
MySQL 5.7과 MySQL 8.0에선 가능하면 임시테이블을 사용하지 않도록 최적화 되었다.**

MySQL 옵티마이저가 내부 쿼리를 외부 쿼리와 병합하는 것이 나을 수도 있고,
때로는 내부 임시 테이블을 생성하는 것이 더 나은 선택일 수 있다.

**이때 MERGE & NO_MERGE 힌트를 통해 옵티마이저의 방법 선택을 유도한다.**

### 9.4.2.9 INDEX_MERGE & NO_INDEX_MERGE

MySQL 서버는 가능하다면 테이블당 하나의 인덱스만을 이용해 쿼리를 처리하려 한다.

단, 하나의 인덱스만으로 검색 대상 범위를 충분히 좁힐 수 없다면 사용 가능한 다른 인덱스를 이용한다.

**여러 인덱스를 통해 검색된 레코드로부터 교집합 또는 합집합만을 구해서 그 결과를 반환할 때,
하나의 테이블에 대해 여러 개의 인덱스를 동시에 사용하는 것을 인덱스 머지(Index Merge)라고 한다.**

인덱스 머지 실행 계획이 항상 성능 향상에 도움이 되는 것은 아니기에,
INDEX_MERGE & NO_INDEX_MERGE 힌트를 통해 인덱스 머지 실행 계획의 사용 여부를 제어한다.

### 9.4.2.10 NO_ICP (NO Index Condition Pushdown)

인덱스 컨디션 푸시다운 최적화는 사용 가능하다면 항상 성능 향상에 도움이 된다.

단, 인덱스 컨디션 푸시다운으로 인해 여러 실행 계획의 실행 계산이 잘못된다면
결과적으로 잘못된 실행 계획을 수립하게 될 수 있기에, **NO_ICP** 를 통한 비활성화 힌트만 지원한다.

### 9.4.2.11 SKIP_SCAN & NO_SKIP_SCAN

**인덱스 스킵 스캔은 인덱스의 선행 칼럼에 대한 조건이 없어도 옵티마이저가 해당 인덱스를 사용할 수 있게 해주는 매우 훌륭한 최적화 기능이다.**

**하지만 조건이 누락된 선행 칼럼이 가지는 유니크한 값의 개수가 많아진다면, 인덱스 스킵 스캔의 성능은 오히려 더 떨어진다.**

MySQL 옵티마이저가 유니크한 값의 개수를 제대로 분석하지 못하거나 잘못된 경로로 인해 비효율적인 인덱스 스킵 스캔을 사용하지 않도록 **NO_SKIP_SCAN** 옵티마이저 힌트를 통해 인덱스 스킵 스캔을 사용하지 않게 할 수 있다.

### 9.4.2.12 INDEX & NO_INDEX

> **INDEX & NO_INDEX 옵티마이저 힌트는 예전 MySQL 서버에서 사용되던 인덱스 힌트의 대체 용도**
>

| **인덱스 힌트** | **옵티마이저 힌트** |
| --- | --- |
| **USE INDEX** | **INDEX** |
| **USE INDEX FOR GROUP BY** | **GROUP_INDEX** |
| **USE INDEX FOR ORDER BY** | **ORDER_INDEX** |
| **IGNORE INDEX** | **NO_INDEX** |
| **IGNORE INDEX FOR GROUP BY** | **NO_GROUP_INDEX** |
| **IGNORE INDEX FOR ORDER BY** | **NO_ORDER_INDEX** |

인덱스 힌트는 특정 테이블 뒤에 사용했기 때문에 별도로 힌트 내에 테이블명 없이 인덱스 이름만 나열했다.

옵티마이저 힌트에는 테이블 명과 인덱스 이름을 함꼐 명시해야 한다.

**아래는 인덱스 힌트를 사용하는 경우와 옵티마이저 힌트를 사용하는 경우의 예시 코드이다.**

```sql
-- // 인덱스 힌트 사용
EXPLAIN
SELECT *
FROM employees USE INDEX(ix_firstname)
WHERE firstname='Matt';
	
-- // 옵티마이저 힌트 사용
EXPLAIN
SELECT /*+ INDEX(employees ix_firstname) */ *
FROM employees
WHERE firstname='Matt';
```