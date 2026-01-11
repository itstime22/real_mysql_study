# **9.3 고급 최적화**

- MySQL 서버의 옵티마이저가 실행 계획 수립 시 통계 정보와 옵티마이저 옵션을 결합해서 최적의 실행 계획을 수립하게 된다.
- 옵티마이저 옵션은 크게 조인 관련된 옵티마이저 옵션과 옵티마이저 스위치로 구분할 수 있다.
- 조인 관련 옵션은 초기 버전부터 존재했지만, 많은 사람이 그다지 신경쓰지 않는 편이다.
    - 조인이 많이 사용되는 서비스에서는 알아야하는 부분이기도 하다.
- **옵티마이저 스위치는** MySQL 5.5 버전부터 지원되기 시작했다.
    - **MySQL 서버의 고급 최적화 기능들을 활성화할지를 제어하는 용도로 사용된다.**

## **9.3.1 옵티마이저 스위치 옵션 내용**

> 옵티마이저 스위치 옵션은 `optimizer_switch` 시스템 변수를 이용해서 제어한다.
`optimizer_switch` 시스템 변수에는 여러 개의 옵션을 세트로 묶어서 설정하는 방식으로 사용한다.
`optimizer_switch` 시스템 변수에 설정 가능한 최적화 옵션은 아래와 같다.
>

![](https://velog.velcdn.com/images/kmw89891/post/a846b2c0-fe27-43c1-85ac-48a1b9b962dc/image.png)

**각각의 옵티마이저 스위치 옵션은 `default`, `on`, `off` 중 하나를 설정 가능하다.**

- `on`으로 설정 시 해당 옵션을 활성화하고, `off`로 설정 시 해당 옵션을 비활성화한다.
- `default`설정 시 기본값이 적용된다.

**옵티마이저 스위치 옵션은 글로벌과 세션별 모두 설정할 수 있는 시스템 변수이므로, MySQL 서버 전체적으로 또는 현재 커넥션에 대해서만 다음과 같이 설정할 수 있다.**

```sql
-- // MySQL 서버 전체적으로 옵티마이저 스위치 설정
SET GLOBAL optimizer_switch='index_merge=on,index_merge_union=on, ...'

-- // 현재 커넥션의 옵티마이저 스위치만 설정
SET SESSION optimizer_switch='index_merge=on,index_merge_union=on, ...'
```

**또는 아래처럼 `SER_VAR` 옵티마이저 힌트를 이용해 현재 쿼리에만 설정할 수도 있다.**

```sql
SELECT /*+ SET_VAR(optimizer_switch='index_merge=on,index_merge_union=on, ...') */
...
FROM ...
```

## 9.3.1.1 MRR과 배치 키 액세스(`mrr` & `batched_key_access`)

> MRR ==  Multi-Range Read == DS-MRR — Disk Sweep Multi-Range Read
>

기존에 지원하던 조인 방식은 드라이빙 테이블의 레코드를 한 건 읽었을 때 드리븐 테이블의 일치하는 레코드를 찾아서 조인을 수행하는 것 (Nested Loop Join)

```java
// 아래처럼 마치 중첩된 반복 명령을 사용하는 것 처럼 작동하기에 Nested Loop Join
for(row1 IN employees){
	for(row2 IN salaries){
		if(condition_matched) return (row1, row2);
	}
}
```

- MySQL 서버 구조상 조인 처리는 MySQL 엔진이, 레코드 검색 및 읽기는 스토리지 엔진이 담당함
- 이때 드라이빙 테이블의 레코드 건별로 각각 드리븐 테이블의 레코드를 찾기에, 레코드를 찾고 읽는 스토리지 엔진에서는 아무런 최적화를 수행할 수 없다.

위 문제점 해결을 위해, **드라이빙 테이블의 레코드가 조인 버퍼에 가득 찼을 때 버퍼링된 레코드를 스토리지 엔진에 한 번에 요청**하도록 할 수 있다.

- **이를 통해 스토리지 엔진이 정렬된 순서로 접근하여 디스크의 페이지 읽기를 최소화** 할 수 있다.
- 이 방식을 MRR(Multi-Range Read)라 한다.
- MRR을 응용해서 실행되는 조인 방식을 BKA(Batched Key Access) 조인이라고 한다.
- **쿼리 특성에 따라 BKA 조인이 큰 도움이 될 수도, 부가적인 정렬 작업에 따라 성능이 저하될 수도 있다.**

## 9.3.1.2 블록 네스티드 루프 조인(`block_nested_loop`)

네스티드 루프 조인과의 가장 큰 차이는 **조인 버퍼의 사용 여부**와 **드라이빙/드리븐 테이블의 조인 순서** 이다.

조인 알고리즘에서 `Block` 이라는 단어 사용 시 조인용으로 별도의 버퍼가 사용됐다는 것을 의미한다.

- 조인 쿼리 실행 계획에서 `Extra` 칼럼에 `Using Join buffer` 라는 문구 표시 시 조인 버퍼를 사용

조인은 드라이빙 테이블에서 일치하는 레코드 건수만큼 드리븐 테이블을 검색하면서 처리된다.

- 드라이빙 테이블은 한 번에 쭉 읽지만, 드리븐 테이블은 여러 번 읽는다는 것을 의미
- 예를들어 드라이빙 테이블 일치 레코드가 1000건 일 때, 드리븐 테이블의 조인 조건이 인덱스를 이용할 수 없다면 드리븐 테이블에서 연결되는 레코드를 찾기 위해 1000번의 풀 테이블 스캔을 해야한다.
- 옵티마이저는 이 경우를 최대한 피하지만, 부득이한 경우 드라이빙 테이블에서 읽은 레코드를 메모리에 캐시한 후 드리븐 테이블과 이 메모리 캐시를 조인하는 형태로 처리한다..
- 이때 사용되는 메모리의 캐시를 조인 버퍼(Join Buffer)라고 한다.

**실제 실행 계획과 반대로, 드리븐 테이블을 먼저 읽고, 위 조인 버퍼에서 일치하는 레코드를 찾는 방식으로 로직이 처리된다.**

- MySQL 8.0.18 버전 해시 조인 알고리즘이 도입되어, 8.0.20 버전부터는 블록 네스티드 루프 조인은 더 이상 사용되지 않고, 해시 조인 알고리즘이 대체되어 사용된다.

## 9.3.1.3 인덱스 컨디션 푸시다운(`index_condition_pushdown`)

`ALTER TABLE employees ADD INDEX ix_lastname_firstname (last_name, first_name);`

처럼 인덱스를 생성했다고 생각해보자.

`SELECT * FROM employees WHERE last_name=’Acton’ AND first_name LIKE ‘%sal’;`

**실행 계획 확인 시 `Extra`에 `Using where` 을 확인할 수 있는데, InnoDB 스토리지 엔진이 읽어서 반환해준 레코드가 인덱스를 사용할 수 없는 WHERE 조건에 일치하는지 검사하는 과정을 의미한다.**

위 쿼리의 경우 first_name LIKE ‘%sal’ 이 인덱스를 사용할 수 없다.

![img_9313.png](img_9313.png)

인덱스 컨디션 푸시다운 기능이 없다면 위 그림처럼 드라이빙 테이블의 일치 레코드에 대해 일일히 확인을 진행 해주어야 하는데, 일치하지 않는 드리븐 테이블의 레코드가 많아질수록 비효율적이다.

근데 우리는 `INDEX ix_lastname_firstname (last_name, first_name)` 를 생성했었다.
이미 앞 단에서 사용한 인덱스에 `first_name` 정보가 포함되어 있었다는 것!

**인덱스 컨디션 푸시다운은 인덱스에 포함된 칼럼의 조건이 있다면 모두 같이 모아서 스토리지 엔진으로 전달**할 수 있게 해주는 핸들러 API의 개선 기능이다.

인덱스를 이용해 최대한 필터링까지 완료해 아래 그림처럼 필요한 레코드 1건에 대해서만 읽기를 수행한다.

**이는 쿼리 성능이 몇 배에서 몇십 배로 향상될 수도 있는 중요한 기능이다.**

## 9.3.1.4 인덱스 확장(`use_index_extensions`)

> **세컨더리 인덱스에 자동으로 추가된 PK를 활용할 수 있게 할지를 결정하는 옵션
(**InnoDB 스토리지 엔진을 사용하는 테이블에서)
>

**`세컨더리 인덱스에 자동으로 추가된 프라이머리 키`** 란?

```sql
CREATE TABLE dept_emp (
	emp_no INT NOT NULL,
	dept_no CHAR(4) NOT NULL,
	from_date DATE NOT NULL,
	to_date DATE NOT NULL,
	PRIMARY_KEY (dept_no,emp_no),
	KEY ix_fromdate (from_date)
);
```

위와 같은 테이블에서, PK는 `(dept_no,emp_no)` 이며 세컨더리 인덱스는 `from_date` 칼럼만 포함한다.

그런데 `ix_fromdate` 는 데이터 레코드를 찾아가기 위해 PK인 dept_no와 emp_no를 순서대로(PK에 명시된 순서) 포함한다.

그래서 **최종적으로 `ix_fromdate` 세컨더리 인덱스가 `(from_date,dept_no,emp_no)` 조합으로 인덱스를 생성한 것과 흡사하게 작동**할 수 있게 된다.

예전 버전에는 자동 추가되는 PK를 인지하지 못했지만, MySQL 서버가 업그레이드 되면서 옵티마이저가 `ix_fromdate` 인덱스의 마지막에 PK가 숨어 있다는 것을 인지하고 실행 계획을 수립하도록 개선되었다.

## 9.3.1.5 인덱스 머지(`index_merge`)

> 인덱스를 이용한 쿼리 실행 시
대부분 옵티마이저는 테이블 별로 하나의 인덱스만 사용하도록 실행 계획을 수립한다.
>

**인덱스 머지 실행 계획을 사용하면 하나의 테이블에 2개 이상의 인덱스를 이용해 쿼리를 처리한다.**

쿼리에서 한 테이블에 대한 WHERE 조건이 여러 개 있더라도 하나의 인덱스에 포함된 칼럼에 대한 조건만으로 인덱스를 검색하고 나머지 조건은 읽어온 레코드에 대해서 체크하는 형태로만 사용되는 것이 일반적이다.

쿼리에 사용된 각각의 조건이 서로 다른 인덱스를 사용할 수 있고 그 조건을 만족하는 레코드 건수가 많을 것으로 예상될 때, MySQL 서버는 인덱스 머지 실행 계획을 선택한다.

**인덱스 머지 실행 계획은 다음과 같이 3개의 세부 실행 계획으로 나누어 볼 수 있다.**

3가지 최적화 모두 여러 개의 인덱스를 통해 결과를 가져온다는 것은 동일하지만, 결과 병합 방식이 다르다.

1. `index_merge_intersection`
2. `index_merge_sort_union`
3. `index_merge_union`

`index_merge` 옵션은 위에 나열된 3개의 최적화 옵션을 한 번에 모두 제어할 수 있는 옵션이다.
아래에서는 3가지 각각의 최적화 방식에 대해 알아보자.

## 9.3.1.6 인덱스 머지 - 교집합(`index_merge_intersection`)

**실행 계획의 Extra 칼럼에 “Using Intersect”라 표시된 경우, 해당 쿼리가 여러개의 인덱스를 각각 검색해서 그 결과의 교집합만 반환했다는 것을 의미한다.**

```sql
// ix_firstname 존재
SELECT *
FROM employees
WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;
```

`Extra` - `Using Intersect(ix_firstname,PRIMARY); Using where;`

`first_name` 조건과 `emp_no` 칼럼의 조건 중 하나라도 효율적인 쿼리 처리가 가능했다면
옵티마이저가 2개의 인덱스를 모두 사용하는 실행 계획을 사용하지 않았을 것이다.

즉 옵티마이저가 각각의 조건에 일치하는 레코드 건수를 예측해 본 결과, 두 조건 모두 상대적으로 많은 레코드를 가져와야 한다는 것을 알게 된 것.

두 조건에 일치하는 값 자체가 적을수록, 각각의 인덱스의 결과 레코드를 하나씩 탐색하는 것은 비효율적인 작업이 될 것이다.

그런데 `ix_firstname` 인덱스는 PK인 `emp_no` 를 자동으로 포함할 것이기에, 인덱스 교집합 보다 `ix_firstname` 자체의 사용이 효율적일 수 있다.

이때는 `index_merge_intersection` 를 비활성화하자(현재 커넥션 또는 쿼리에 대해서만 제어도 가능하다.)

## 9.3.1.7 인덱스 머지 - 합집합(`index_merge_union`)

**인덱스 머지(`type`=`index_merge`)의 `Using union` 은 WHERE 절에 사용된 2개 이상의 조건이 각각의 인덱스를 사용하되, `OR` 연산자로 연결된 경우에 사용되는 최적화이다.**

`ix_firstname` 과 `ix_hiredate` 가 있고 `WHERE first_name=? OR hire_date=?` 라 가정했을때,

**인덱스 머지 최적화가 `ix_firstname` 결과와 `ix_hiredate` 결과를 Union 알고리즘으로 병합했다는 것**

**`Union` 알고리즘 특성상 중복된 정보가 생성될 수 있는데, 각각의 인덱스 결과값은 PK로 정렬되어있기에, 순서대로 읽어 중복 제거를 진행하는데, 이때 Priority Queue 방식을 사용한다.**

> OR 조건은 두 조건 중 하나라도 제대로 인덱스를 사용하지 못하면 항상 풀 테이블 스캔으로 처리된다.
>

## 9.3.1.8 인덱스 머지 - 정렬 후 합집합(`index_merge_sort_union`)

OR 연산에서 특정 쿼리의 결과가 PK로 정렬되어 있지 않은 경우, 중복 제거를 위해 우선순위 큐를 사용하는 것이 불가능하기에, 각 집합을 PK로 정렬한 다음 중복 제거를 수행한다.

실행계획은 `Extra` - `Using sort_union(ix_firstname,ix_hiredate)` 처럼 표기된다.

## 9.3.1.9 세미 조인(`semijoin`)

> 다른 테이블과 실제 조인을 수행하지는 않고, 단지 다른 테이블에서 조건에 일치하는 레코드가 있는지만 체크하는 형태의 쿼리를 세미 조인 이라고 한다.
**메인 쿼리에서 사용된 테이블과 서브쿼리 결과를 조인하는 조인**
>

```sql
SELECT *
FROM employees e
WHERE e.emp_no IN
	(SELECT de.emp_no FROM dept_emp de WHRER de.from_date='1995-01-01');
```

예전에는 위와 같은 쿼리에 대해 서브쿼리 부분을 먼저 진행하는 것이 아닌, employees 테이블을 풀 스캔하며 각각의 레코드에 대해 비교를 진행했다. 57건으로 끝날 것을 30만건 넘게 읽어서 처리했던 것.

### 세미 조인 VS 안티 세미 조인

세미 조인 형태의 쿼리와 안티 세미 조인 형태의 쿼리는 최적화 방식에 조금 차이가 있다.

**세미 조인 형태 (**`=` 또는 `IN`)

- **세미 조인 최적화**
- IN-to-EXISTS 최적화
- MATERIALIZATION 최적화

**안티 세미 조인 형태 (`<>`** 또는 `NOT IN`)

- IN-to-EXISTS 최적화
- MATERIALIZATION 최적화

### 세미 조인 최적화

> 최근 도입된 세미 조인 최적화에 대해서만 알아보면 다음과 같다.
>
- **`Table Pull-out`**
- **`Duplicate Weed-out`**
- **`First Match`**
- **`Loose Scan`**
- **`Materialization`**

쿼리에 사용되는 테이블과 조인 조건 특성에 따라 옵티마이저는 각 전략을 선별적으로 사용한다.

`Table Pull-out` 은 사용 가능하면 항상 세미 조인보다 좋은 성능을 내기에 별도 설정이 없다.

**`First Match` 와 `Loose Scan` 은 각각 `firstmatch` 와 `loosescan` 옵션으로 사용 여부를 결정하며,
`Duplicate Weed-out` 과 `Materialization` 최적화 전략은 `materialization` 옵티마이저 스위치로 사용 여부를 선택할 수 있다.**

`optimizer_switch` 시스템 변수의 semijoin 옵티마이저 옵션은 `firstmatch` 와 `loosescan`, `materialization` 옵티마이저 옵션을 한 번에 활성화 하거나 비활성화 할 때 사용한다.

## 9.3.1.10 테이블 풀-아웃(Table Pull-out)

> 세미 조인의 서브쿼리에 사용된 테이블을 아우터 쿼리로 끄집어낸 후에 쿼리를 조인 쿼리로 재작성하는 형태의 최적화
>

```sql
// IN 형태의 세미 조인이 가장 빈번하게 사용되는 형태는 아래와 같다.
EXPLAIN
SELECT * FROM employees e
WHERE e.emp_no IN (SELECT de.emp_no FROM dept_emp de WHERE de.dept_no='d009');
```

Table pullout 최적화는 모든 형태의 서브쿼리에서 사용될 수 있는 것은 아니기에, 몇 가지 제한 사항과 특성을 살펴보자

- Table pullout 최적화는 세미 조인 서브 쿼리에서만 사용 가능하다
- Table pullout 최적화는 서브쿼리 부분이 UNIQUE 인덱스나 PK 룩업으로 결과가 1건인 경우에만 사용 가능하다.
- Table pullout 최적화가 적용된다고 하더라도, 기존 쿼리에서 가능했던 최적화 방법이 사용 불가능한 것은 아니므로, MySQL에서는 가능하다면 Table pullout 최적화를 최대한 적용한다.
- **Table pullout 최적화는 서브쿼리의 테이블을 아우터 쿼리로 가져와 조인으로 풀어쓰는 최적화를 수행**하는데, 만약 서브쿼리의 모든 테이블이 아우터 쿼리로 끄집어낼 수 있다면 서브쿼리 자체는 없어진다.
- MySQL에서는 “최대한 서브쿼리를 조인으로 풀어서 사용해라”라는 튜닝 가이드가 많은데, Table pullout 최적화는 이 가이드를 그대로 실행하는 것이라 서브쿼리를 조인으로 풀어서 사용할 필요가 없다

## 9.3.1.11 퍼스트 매치(firstmatch)

> **퍼스트 매치 최적화 전략은 `IN(subquery)` 형태의 세미조인을 `EXISTS(subquery)` 형태로 튜닝한 것과 비슷한 방법으로 실행된다.**
>

실행 계획에서 ID 값이 모두 1이라는건 서브 쿼리가 아닌 조인으로 처리되었다는 것.

> **`id`는 “쿼리 내에서 실행되는 SELECT 단위”를 구분하는 식별자**
>
> - 하나의 `SELECT` 문 → 보통 `id = 1`
> - **서브쿼리 / UNION / 파생 테이블**이 있으면 → `id`가 여러 개로 나뉨
> - `id` 값이 **클수록 먼저 실행**됨

**테이블의 레코드에 대해 조건에 일치하는 레코드 1건만 찾으면 더이상의 테이블 검색을 하지 않는 것.**

실제 의미론적으로는 EXISTS(subquery)와 동일하게 처리되는 것이다.

`IN-to-EXISTS` 최적화도 유사한 방식으로 처리되지만, 아래와 같은 퍼스트 매치의 이점이 있다.

1. 가끔 여러 테이블이 조인되는 경우 원래 쿼리에 없던 동등 조건을 옵티마이저가 자동 추가해준다.
   이러한 최적화가 기존 `IN-to-EXISTS`에서는 서브쿼리 내에서만 진행되었지만,
   `FirstMatch`에서는 조인 형태로 처리되어 최적화 적용 전파 범위가 넓어진다.
2. `IN-to-EXISTS` 에선 아무런 조건 없이 변환이 가능한 경우 무조건 최적화를 수행하는데,
   `FirstMatch` 에서는 서브쿼리의 모든 테이블에 대해 최적화를 수행할지, 일부 테이블만 수행할지 취사 선택이 가능해진다.

**FirstMatch 최적화는 아래와 같은 제한 사항과 특성을 가진다.**

- 서브쿼리는 그 서브쿼리가 참조하는 모든 아우터 테이블이 먼저 조회된 이후에 실행된다.
- FirstMatch 최적화 사용 시 실행계획 Extra 칼럼에 FirstMatch(table-N) 문구가 표시된다.
- FirstMatch 최적화는 상관 서브쿼리에서도 사용될 수 있다.
- FirstMatch 최적화는 GROUP BY나 집합 함수가 사용된 서브쿼리의 최적화에는 사용될 수 없다.

FirstMatch 최적화는 `optimizer_switch` 시스템 변수에서 `semijoin` 옵션과 `firstmatch` 옵션이 모두 ON으로 활성화된 경우에만 사용할 수 있다.

## 9.3.1.12 루스 스캔(loosescan)

> **루스 인덱스 스캔**
집계 함수가 존재하는 쿼리에서 인덱스 리프 노드에서 불필요한 부분은 무시하며 스캔을 진행하는 방식
>

세미 조인 서브쿼리 최적화의 LooseScan은 인덱스를 사용하는 GROUP BY 최적화 방법에서 살펴본
`Using Index for group-by` 의 루스 인덱스 스캔과 비슷한 읽기 방법을 사용한다.

```sql
EXPLAIN
SELECT * FROM departments d 
WHERE d.dept_no 
	IN (SELECT de.dept_no FROM dept_emp de );
```

https://oopy.lazyrockets.com/api/v2/notion/image?src=https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F30ad3a6f-0153-45d5-9923-45374fde1649%2FUntitled.png&blockId=92a90cdf-db0b-45d6-9654-faa0348b53e0

위 그림에서 `dept_emp` 테이블은 `dept_no`+`emp_no` 칼럼의 조합으로 PK 인덱스가 만들어져서,
`dept_no`로 그룹핑 시 결국 9건 밖에 없다는 것을 알 수 있다.

이때 `dept_emp` 테이블의 PK를 루스 인덱스 스캔으로 UNIQUE한 `dept_no` 만 읽으면 효율적인 서브쿼리 실행이 가능해진다.

## 9.3.1.13 구체화(Materialization)

> **Materialization 최적화는 세미 조인에 사용된 서브쿼리를 통째로 구체화해, 쿼리를 최적화한다는 의미**
>

**구체화(Materialization)는 쉽게 말해, 내부 임시 테이블을 생성한다는 것**

e.g., 1995년 1월 1일 조직이 변경된 사원들의 목록 조회

```sql
SELECT *
FROM employees e
WHERE e.emp_no 
	IN (SELECT de.emp_no FROM dept_emp de
			WHERE de.from_date='1995-01-01');
```

이 경우 select type에 MATERIALIZED 가 나타난다. 이 쿼리에서
사용하는 테이블은 2개인데, 실행 계획은 3개 라인이 출력된 것으로 임시 테이블의 생성을 짐작할 수 있다.

**다른 서브쿼리 최적화와 달리 서브쿼리 내에 `GROUP BY` 절이 있어도 이 최적화 전략을 사용할 수 있다.**

> **Materialization 최적화가 사용될 수 있는 형태의 쿼리에도 몇 가지 제한사항과 특성이 존재한다.**
>
1. IN (subquery) 에서 서브쿼리는 상관 서브쿼리가 아니어야 한다.

   > **상관 서브쿼리란? 서브쿼리가 외부 쿼리의 컬럼을 참조하는 경우**
   >
   >
   > orders 한 행 읽음
   > → 서브쿼리 실행
   > orders 다음 행
   > → 서브쿼리 또 실행
   > 와 같은 형태로 진행되기에, **외부 쿼리의 각 행마다 실행되어 대규모 데이터에서 매우 위험**하다.
>
2. 서브쿼리는 GROUP BY나 집합 함수들이 사용돼도 구체화를 사용할 수 있다.
3. 구체화가 사용된 경우에는 내부 임시 테이블이 사용된다.

**Materialization** 최적화는 `optimizer_switch` 시스템 변수에서 `semijoin` 옵션과 `materialization` 옵션이 모드 ON으로 활성화된 경우에만 사용할 수 있다.

## 9.3.1.14 중복 제거(Duplicated Weed-out)

> **세미 조인 서브쿼리를 일반적인 INNER JOIN 쿼리로 바꿔서 실행하고
마지막에 중복된 레코드를 제거하는 방법으로 처리되는 최적화 알고리즘이다.**
>

```sql
-- e.g., 급여가 150000 이상인 사원들의 정보를 조회하는 쿼리
EXPLAIN
SELECT * FROM employees e
WHERE e.emp_no 
	IN (SELECT s.emp_no 
			FROM salaries s 
			WHERE s.salary>150000);
```

salaries 테이블의 PK가 `(emp_no+from_date)` 이기에 중복된 emp_no가 발생할 수 있다.

위 쿼리를 GROUP BY 절을 넣어 재작성하면 기존 세미 조인 서브쿼리와 동일한 결과를 얻을 수 있다.

```sql
SELECT *
FROM employees e, salaries s
WHERE e.emp_no=s.emp_no AND s.salary>150000
GROUP BY e.emp_no;
```

**Duplicated Weed-out은 상단 쿼리의 실행을 하단 쿼리의 실행처럼 처리해준다.**

**실제 처리 과정은 아래와 같다.**

1. `salaries` 테이블의 `ix_salary` 인덱스를 스캔하여 `salary`가 150000 보다 큰 사원을 검색해 `employees` 테이블 조인을 실행
2. 조인된 결과를 임시 테이블에 저장
3. 임시 테이블에 저장된 결과에사 `emp_no` 기준으로 중복 제거
4. 중복을 제거하고 남은 레코드를 최종적으로 반환

실행 계획에서는 `Duplicated Weed-out` 이라는 문구가 별도로 표시되지는 않는다.
대신 `Extra` 칼럼에 `Start temporary`와 `End temporary` 문구가 별도로 표기된다.

**실질적인 처리 과정에서는 1번 2번 과정이 반복적으로 실행된다.**

이 반복과정이 실행되는 테이블의 실행 계획 라인에는 `Start temporary` 문구가,

반복 과정이 끝나는 테이블의 실행 계획 라인에는 `End temporary` 문구가 표시된다.

## 9.3.1.15 컨디션 팬아웃(`condition_fanout_filter`)

> **조인을 실행할 때 테이블의 순서는 쿼리의 성능에 매우 큰 영향을 미친다.**
예를 들어 A 테이블과 B 테이블을 조인할 때
A 테이블은 조건 일치 레코드가 1만건 이고, B 테이블은 10건 이라고 가정해보자.
A 테이블이 조인의 드라이빙 테이블로 결정되면 B 테이블을 1만번 읽어야 한다.
B 테이블의 인덱스를 이용해 조인을 실행한다 해도 레코드를 읽을 때 마다 B 테이블의 인덱스를 구성하는 B-Tree의 루트 노드부터 검색을 실행해야한다.
**MySQL 옵티마이저는 여러 테이블이 조인되는 경우
가능하다면 일치하는 레코드 건수가 적은 순서대로 조인을 실행한다.**
>

```sql
-- employees 테이블에서 이름이 Matt면서 입사일자가 1985-11-21 ~ 1986-11-21 인 사원 급여
SELECT *
FROM employees e
	INNER JOIN salaries s ON s.emp_no=e.emp_no
WHERE e.first_name='Matt'
	AND e.hire_date BETWEEN '1985-11-21' AND '1986-11-21';
```

**`condition_fanout_filter` 를 비활성화 하면 실행 계획은 다음과 같이 나온다.**

| id | table | type | key | rows | filtered | Extra |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | e | ref | ix_firstname | 233 | **100.00** | Using where |
| 1 | s | ref | PRIMARY | 10 | 100.00 | NULL |
1. `employees` 테이블에서 `ix_firstname` 인덱스를 이용해 `first_name=’Matt’` 조건에 일치하는 233건의 레코드를 검색한다.
2. 검색된 233건의 레코드 중 `hire_date` 가 `1985-11-21`와 `1986-11-21` 사이인 레코드를 걸러낸다.

   > 이때 filtered 칼럼의 값이 100인 것은 옵티마이저가 233건 모두
   hire_date 칼럼의 조건을 만족할 것을 예측했다는 것을 의미한다.
>
3. `employees` 테이블을 읽은 결과 233 건에 대해 `salaries` 테이블의 PK를 이용해 `salaries` 테이블의 레코드를 읽는다. 이때 옵티마이저는 `employees` 테이블의 레코드 한 건당 `salaries` 테이블의 레코드 10건이 일치할 것으로 예상했다.

**`condition_fanout_filter` 활성화 시 실행 계획은 다음과 같이 나온다.**

| id | table | type | key | rows | filtered | Extra |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | e | ref | ix_firstname | 233 | **23.20** | Using where |
| 1 | s | ref | PRIMARY | 10 | 100.00 | NULL |

`condition_fanout_filter` 비활성화 때는 employees 테이블에서 모든 조건을 충족하는 레코드가 **233**건 일 것으로 예측한 반면, 활성화 시 **54건(233*0.2320)**만 조건을 충족할 것이라 예측했다.

**옵티마이저가 조건을 만족하는 레코드 건수를 정확하게 예측할 수 있다면 더 빠른 실행 계획을 만들것이다.**

`condition_fanout_filter` 최적화는 어떻게 filtered 칼럼의 값을 예측해낼까?

1. WHERE 조건절에 사용된 칼럼에 대해 인덱스가 있는 경우
2. WHERE 조건절에 사용된 칼럼에 대해 히스토그램이 존재하는 경우

쿼리 실행 계획이 잘못된 선택을 한 적이 별로 없다면 성능에 크게 도움이 되지 않을 수 있다.

실행 계획 수립 자체에도 오버헤드가 생기므로 가능하다면 업그레이드 실행 전 성능 테스트를 진행하자.

## 9.3.1.16 파생 테이블 머지(`derived_merge`)

> 예전 버전에서는 FROM 절에 사용된 서브쿼리를 먼저 실행하여, 임시 테이블로 만들어 처리했다.
>

```sql
EXPLAIN
SELECT * FROM (
	SELECT * FROM employees WHERE first_name='Matt'
) derived_table
WHERE derived_table.hire_date='1986-04-03'; 
```

| id | select_type | table | type | key |
| --- | --- | --- | --- | --- |
| 1 | PRIMARY | <derived2> | ref | <auto_key0> |
| 2 | DERIVED | employees | ref | ix_firstname |

위 실행 계획을 보면 `employees` 테이블을 읽는 라인의 `select_type` 칼럼이 `DERIVED` 로 표기된다.

이는 `employees` 테이블에서 `first_name` 칼럼의 값이 ‘Matt’인 레코드들만 읽어 임시 테이블을 생성하고,
이 임시 테이블을 다시 읽어서 `hire_date` 로 걸러내어 반환한 것이다.

MySQL에서는 이렇게 FROM 절에 사용된 서브쿼리를 파생 테이블(Derived Table)이라고 부른다.

임시 테이블 생성과, 다시 읽는 행위에 오버헤드가 추가된다.

**임시 테이블 자체도 처음엔 메모리에 생성되지만, 레코드 건수가 많아지면 디스크로 기록된다.
이 때문에 크기가 작다면 성능에 영향이 적겠지만, 임시 테이블이 커지면 쿼리의 성능이 많이 저하될 것이다.**

MySQL 5.7 부터는 파생 테이블로 만들어지는 서브쿼리를 외부 쿼리와 병합해서 서브쿼리 부분을 제거하는 최적화가 도입되었는데, `derived_merge` 옵션은 이러한 임시 테이블 최적화를 활성화할지 여부를 결정한다.

| id | select_type | table | type | key |
| --- | --- | --- | --- | --- |
| 1 | SIMPLE | employees | index_merge | ix_hiredate,ix_firstname |

활성화 이후 서브쿼리 대신 외부쿼리를 활용하도록 변경된 것을 확인할 수 있다.

MySQL 옵티마이저가 자동 외부쿼리로의 병합을 지원하지만, 다음과 같은 조건에선 불가하기에 가능하다면 **서브쿼리는 외부쿼리로 수동으로 병합해서 작성**해야 쿼리 성능 향상에 도움이 된다.

1. 집계 함수와 윈도우 함수가 사용된 서브쿼리
2. DISTINCT 가 사용된 서브쿼리
3. GROUP BY나 HAVING 이 사용된 서브쿼리
4. LIMIT이 사용된 서브쿼리
5. UNION 또는 UNION ALL이 사용된 서브쿼리
6. SELECT 절에 사용된 서브쿼리
7. 값이 변경되는 사용자 변수가 사용된 서브쿼리

## 9.3.1.17 인비저블 인덱스(`use_invisible_indexes`)

> MySQL 8.0 부터 인덱스의 가용 상태를 제어할 수 있게 되었다.
>

`ALTER TABLE employees ALTER INDEX ix_hiredate VISIBE/INVISIBE;`

옵티마이저가 `ix_hiredate` 인덱스를 `사용할 수 있게`/`사용하지 못하게` 변경

`SET optimizer_switch=’use_invidible_indexes=on’`
설정을 통해 옵티마이저가 INVISIBLE 상태의 인덱스도 볼 수 있게 할 수 있다.

## 9.3.1.18 스킵 스캔(`skip_scan`)

인덱스의 핵심은 값이 정렬되어 있다는 것 → 인덱스를 구성하는 칼럼의 순서가 중요하다.

(A,B,C) 구성의 인덱스는 A / A&B 조건에는 사용 가능하지만, B나 C에 대해선 활용 불가능하다.

**인덱스 스킵 스캔은 제한적이지만 인덱스의 이런 제약 사항을 뛰어넘을 수 있는 최적화 기법이다.**

인덱스의 선행 칼럼이 조건절에 사용되지 않더라도 후행 칼럼의 조건만으로도 인덱스를 이용한 쿼리 성능 개선이 가능하다. → 마치 선행 조건이 있는 것 처럼 모든 선행 조건 값에 대해 가져온 후 뒤 인덱스를 사용한다.

다만 인덱스의 선행 칼럼이 매우 다양한 값을 가지는 경우 인덱스 스킵 스캔이 비효율적이 되는데,
**MySQL8.0 옵티마이저는 인덱스 선행 칼럼이 소수의 유니크한 값을 가질때만 인덱스 스킵 스캔을 사용한다.**

## 9.3.1.19 해시 조인(`hash_join`)

**MySQL 8.0.18 버전부터는 해시 조인이 추가로 지원되기 시작했다.**

**해시조인은 기존 방식인 네스니트 루프 조인 방식보다 첫 번째 레코드를 찾는 시간은 많이 걸리지만,
최종 레코드를 찾는 데까지는 시간이 많이 걸리지 않는다.**

즉 해시 조인 쿼리는 최고 스루풋(Best Throughput) 전략에 적합하며,
네스티드 루프 조인은 최고 응답 속도(Best Response-time) 전략에 적합하다

일반적인 웹 서비스는 **온라인 트랜잭션(OLTP) 서비스이기에** 스루풋도 중요하지만 **응답 속도가 더 중요**하다.
분석과 같은 서비스는 사용자의 응답 시간보다는 **전체적으로 처리 소요 시간이 중요하기에 스루풋**을 본다.

그래서 일반적인 웹 서비스 DB로 사용되는 MySQL 에서는 여전히 네스티드 루프 조인 방식을 사용한다.

다만 MySQL 8.0.20 버전 이후,
안티 조인이나 세미 조인에 사용되던 블록 네스티드 루프 조인 대신 해시 조인을 사용되도록 바뀌었다.

### 해시 조인 처리 방식

> 일반적으로 해시 조인은 **빌드 단계**와 **프로브 단계**로 나뉘어 처리된다.
>

빌드 단계에서는 조인 대상 테이블 중에서 레코드 건수가 적어서 해시 테이블로 만들기에 용이한 테이블을 골라 메모리에 해시 테이블을 생성(빌드)하는 작업을 수행한다.

- 빌드 단계에서 해시 테이블을 만들 때 사용되는 원본 테이블을 빌드 테이블이라고도 한다.

프로브 단계는 나머지 테이블의 레코드를 읽어서 해시 테이블의 일치 레코드를 찾는 과정을 의미한다.

- 이때 읽는 나머지 테이블을 프로브 테이블이라고도 한다.

실행 계획의 포맷을 `TREE` 로 설정하여 보았을 때,

```sql
EXPLAIN FORMAT=TREE
SELECT ...

-> Inner hash join
	-> Table scan on e
	-> Hash
		-> Table scan on de
```

최하단 제일 안쪽(들여쓰기가 가장 많이 된)의 dept_emp 테이블이 빌드 테이블로 선정 됨을 알 수 있다.

해시 테이블을 메모리에 저장 시 MySQL 서버는 `join_buffer_size` 시스템 변수로 크기를 제어할 수 있는 조인 버퍼를 사용하는데, 해시 테이블의 레코드 건수가 많아 조인 버퍼의 공간이 부족한 경우 적당한 크기의 청크로 분리한 다음 아래 그림과 같은 방식으로 해시 조인을 처리한다.

![](https://infoqoch.github.io/assets/image/2024-06-26-mysql%20optimizer/hash2.png)

## 9.3.1.20 인덱스 정렬 선호(`prefer_ordering_index`)

> MySQL 옵티마이저는 `ORDER BY` 또는 `GROUP BY`를 인덱스를 사용해 처리 가능한 경우
쿼리의 실행 계획에서 이 인덱스의 가중치를 높이 설정해서 실행된다.
>

옵티마이저의 실행 계획 실수가 자주 발생할 경우 `prefer_ordering_index` 를 `ON` 에서 `OFF` 로 변경하여 `ORDER BY`를 위한 인덱스에 너무 가중치를 부여하지 않도록 옵티마이저 옵션을 설정할 수 있다.