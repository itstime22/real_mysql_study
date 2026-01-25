# 9.3.2 조인 최적화 알고리즘

MySQL에는 조인 쿼리의 실행 계획(Execution Plan)을 최적화하기 위한 **조인 순서 탐색 알고리즘**이 존재합니다.

조인 대상 테이블 수가 늘어나면 가능한 조인 순서(탐색 공간)가 **팩토리얼(N!)**로 폭증하여, 최적 실행 계획 탐색이 매우 어려워집니다.

```sql
SELECT *
FROM t1, t2, t3, t4
WHERE ...
```

---

## 9.3.2.1 Exhaustive 검색 알고리즘

MySQL 5.0과 그 이전에서 사용되던 방식입니다.

### 특징

- FROM 절에 명시된 **모든 테이블 조합(조인 순서)에 대해 실행 비용을 계산**
- 그중 **최소 비용인 실행 계획을 선택**

### 문제점

| 테이블 개수 | 경우의 수 (N!) | 비고                            |
| ----------- | -------------- | ------------------------------- |
| 3개         | 6              | -                               |
| 4개         | 24             | -                               |
| 5개         | 120            | -                               |
| 10개        | 3,628,800      | 실행 계획 수립 시간 급격히 증가 |
| 20개        | 2.43 × 10¹⁸    | 현실적으로 탐색 불가능          |

- 테이블이 **N개면 경우의 수는 N!**
- N이 커질수록 비용 계산 횟수가 폭증 → **실행 계획 수립만으로도 지연이 커질 수 있음**
- 테이블이 **10개를 넘어가면 실행 계획 수립에 시간이 급격히 증가**

---

## 9.3.2.2 Greedy 검색 알고리즘

MySQL 5.0 이후 조인 최적화에서 현실적인 탐색을 위해 사용되는 접근 방식입니다.

### optimizer_search_depth

이 파라미터는 탐색 깊이를 제어합니다.

| 값       | 의미                                   |
| -------- | -------------------------------------- |
| **1~62** | 지정된 깊이만큼만 탐색                 |
| **0**    | 옵티마이저가 자동으로 탐색 깊이를 결정 |

**설정 확인:**

```sql
SHOW VARIABLES LIKE 'optimizer_search_depth';
```

**설정 변경:**

```sql
SET optimizer_search_depth = 4;  -- 4개 테이블까지만 탐색
```

### Greedy 알고리즘 절차

다음과 같은 단계로 최적 조인 순서를 찾습니다:

1. 전체 N개 테이블 중에서 `optimizer_search_depth` 만큼의 테이블 조합을 만들어 **실행 비용 계산**
2. **가장 비용이 작은 실행 계획을 하나 선택**
3. 선택된 실행 계획의 첫 번째 테이블을 **"부분 실행 계획(partial plan)"의 첫 테이블로 고정**
4. 남은 테이블(N-1개) 중에서 다시 `optimizer_search_depth`만큼 조합을 만들고
5. 그 조합들을 partial plan 뒤에 붙여보며 **비용을 비교**
6. **가장 좋은 후보를 두 번째 테이블로 고정**
7. **남은 테이블이 없어질 때까지 반복**
8. 최종적으로 **partial plan의 테이블 순서가 조인 순서가 됨**

```
초기: [t1, t2, t3, t4] (4개 테이블)
1단계: [t1, t2] 조합 선택 → t1 고정
2단계: [t2, t3, t4] 중에서 t1 뒤에 올 최적 테이블 선택 → t2 고정
3단계: [t3, t4] 중에서 선택 → t3 고정
4단계: 남은 t4 추가
최종: t1 → t2 → t3 → t4
```

### optimizer_prune_level

조인 순서 최적화에서 휴리스틱 기반 가지치기(pruning)를 제어합니다.

| 값    | 의미                                 |
| ----- | ------------------------------------ |
| **1** | 휴리스틱 기반 가지치기 적용 (기본값) |
| **0** | 휴리스틱 최적화 비활성화             |

**설정 확인:**

```sql
SHOW VARIABLES LIKE 'optimizer_prune_level';
```

> 💡 **권장**: 특별한 이유가 없다면 기본값(1) 유지가 일반적입니다.

---

## 9.4 쿼리 힌트

MySQL 옵티마이저는 통계/규칙 기반으로 실행 계획을 세우지만, 서비스 특성이나 데이터 분포를 100% 정확히 이해하지 못할 수 있습니다.

그래서 개발자가 옵티마이저에게 "이렇게 실행 계획을 잡아줘"라고 방향을 제시할 수 있는 수단이 필요하며, 그 수단이 힌트(Hint)입니다.

### 힌트의 종류

MySQL에서 사용 가능한 힌트는 크게 2가지입니다:

| 힌트 유형           | 형식                            | 특징            |
| ------------------- | ------------------------------- | --------------- |
| **인덱스 힌트**     | `STRAIGHT_JOIN`, `USE INDEX` 등 | SQL 구문에 포함 |
| **옵티마이저 힌트** | `/*+ ... */` 형태               | 주석 기반       |

> 💡 **권장**: 가능하면 **옵티마이저 힌트 사용을 권장**  
> → 다른 DB에서는 주석으로 무시되기 쉬워 이식성(ANSI-SQL 관점) 측면에서 유리합니다.

---

## 9.4.1 인덱스 힌트

인덱스 힌트는 SQL 문법 요소로 들어가므로, **MySQL 의존성이 강해지고 이식성이 떨어질 수 있습니다**.

### 9.4.1.1 STRAIGHT_JOIN

여러 테이블 조인 시, **조인 순서를 고정**합니다. (작성한 FROM 절 테이블 순서를 그대로 따르도록 유도)

```sql
SELECT STRAIGHT_JOIN
  e.first_name, e.last_name, d.dept_name
FROM employees e, dept_emp de, departments d
WHERE e.emp_no = de.emp_no
  AND d.dept_no = de.dept_no;
```

- 위 예시는 **e → de → d 순서를 강제**

**유사한 옵티마이저 힌트:**

- `JOIN_FIXED_ORDER`: 동일 목적(순서 고정)
- `JOIN_ORDER` / `JOIN_PREFIX` / `JOIN_SUFFIX`: 일부 순서만 가이드

### 9.4.1.2 USE INDEX / FORCE INDEX / IGNORE INDEX

특정 테이블에서 어떤 인덱스를 쓰거나/쓰지 말지 지정합니다. 힌트는 해당 테이블 뒤에 작성합니다.

#### 힌트 종류

| 힌트             | 의미                                        | 강제 수준                               |
| ---------------- | ------------------------------------------- | --------------------------------------- |
| **USE INDEX**    | "이 인덱스를 쓰는 방향으로 고려해줘" (권장) | 약함 (대부분 따르지만 항상 강제는 아님) |
| **FORCE INDEX**  | USE INDEX보다 더 강한 유도                  | 강함                                    |
| **IGNORE INDEX** | 특정 인덱스를 쓰지 못하게 함                | 강제                                    |

```sql
-- USE INDEX
SELECT *
FROM employees USE INDEX(ix_firstname)
WHERE firstname = 'Matt';

-- FORCE INDEX
SELECT *
FROM employees FORCE INDEX(ix_firstname)
WHERE firstname = 'Matt';

-- IGNORE INDEX
SELECT *
FROM employees IGNORE INDEX(ix_firstname)
WHERE firstname = 'Matt';
```

#### 용도 제한

인덱스 힌트는 특정 용도로 제한할 수 있습니다:

- `USE INDEX FOR JOIN`: 조인에만 사용
- `USE INDEX FOR ORDER BY`: 정렬에만 사용
- `USE INDEX FOR GROUP BY`: 그룹화에만 사용

### 9.4.1.3 SQL_CALC_FOUND_ROWS

LIMIT이 있는 쿼리는 원하는 건수만 찾으면 보통 탐색을 중단합니다. 하지만 `SQL_CALC_FOUND_ROWS`를 쓰면 LIMIT을 만족해도 끝까지 탐색하여 "LIMIT이 없었다면 총 몇 건인지"를 알아낼 수 있습니다.

```sql
SELECT SQL_CALC_FOUND_ROWS *
FROM employees
WHERE firstname = 'Matt'
LIMIT 10;

SELECT FOUND_ROWS();  -- LIMIT 없이 조회했을 때의 총 건수
```

---

## 9.4.2 옵티마이저 힌트

### 9.4.2.1 옵티마이저 힌트의 영향 범위

옵티마이저 힌트는 다양한 범위에 적용할 수 있습니다:

| 범위                  | 설명                    | 예시                                   |
| --------------------- | ----------------------- | -------------------------------------- |
| **인덱스 레벨**       | 특정 인덱스 지정 가능   | `/*+ INDEX(employees ix_firstname) */` |
| **테이블 레벨**       | 특정 테이블 대상        | `/*+ INDEX(employees ix_firstname) */` |
| **쿼리 블록 레벨**    | 해당 쿼리 블록에만 적용 | 서브쿼리 내부                          |
| **글로벌(쿼리 전체)** | 전체 쿼리에 영향        | `/*+ MAX_EXECUTION_TIME(100) */`       |

```sql
EXPLAIN
SELECT /*+ INDEX(employees ix_firstname) */
FROM employees
WHERE firstname = 'Matt';
```

### 9.4.2.2 MAX_EXECUTION_TIME

실행 계획에는 영향을 주지 않고, **실행 시간 상한만 설정**합니다.

```sql
SELECT /*+ MAX_EXECUTION_TIME(100) */ *
FROM employees
WHERE firstname = 'Matt';
```

**특징:**

- 단위: **밀리초(ms)**
- 제한 시간 초과 시 쿼리 중단 오류 발생

### 9.4.2.3 SET_VAR

특정 쿼리에서만 **일시적으로 시스템 변수**(특히 `optimizer_switch`) 등을 변경해 실행 계획에 영향을 주는 힌트입니다.

```sql
EXPLAIN
SELECT /*+ SET_VAR(optimizer_switch='index_merge_intersection=off') */ *
FROM employees
WHERE firstname = 'Matt';
```

**활용:**

```sql
-- 조인 버퍼 크기 일시적 증가
SELECT /*+ SET_VAR(join_buffer_size=1M) */ *
FROM large_table1 t1
INNER JOIN large_table2 t2 ON t1.id = t2.id;

-- 정렬 버퍼 크기 일시적 증가
SELECT /*+ SET_VAR(sort_buffer_size=2M) */ *
FROM large_table
ORDER BY col1, col2, col3;
```

### 9.4.2.4 SEMIJOIN / NO_SEMIJOIN

세미조인 최적화(서브쿼리 IN/EXISTS 변환 등)의 세부 전략을 제어합니다.

| 전략                   | 힌트                        | 설명                        |
| ---------------------- | --------------------------- | --------------------------- |
| **Duplicate Weed-out** | `SEMIJOIN(DUPSWEEDOUT)`     | 중복 제거 방식              |
| **First Match**        | `SEMIJOIN(FIRSTMATCH)`      | 첫 번째 매치만 찾기         |
| **Loose Scan**         | `SEMIJOIN(LOOSESCAN)`       | 느슨한 스캔                 |
| **Materialization**    | `SEMIJOIN(MATERIALIZATION)` | 임시 테이블 생성            |
| **Table Pull-out**     | 별도 힌트 없음              | 가능하면 유리해서 자동 적용 |

```sql
SELECT /*+ SEMIJOIN(FIRSTMATCH) */ *
FROM employees e
WHERE e.emp_no IN (
    SELECT de.emp_no
    FROM dept_emp de
    WHERE de.dept_no = 'd001'
);
```

### 9.4.2.5 SUBQUERY

세미조인 최적화가 적절히 적용되지 못할 때, 서브쿼리 최적화 방향을 제어하는 힌트입니다. (특히 안티 세미조인 상황에서 유용할 수 있음)

| 최적화 방법         | 힌트                        |
| ------------------- | --------------------------- |
| **IN-to-EXISTS**    | `SUBQUERY(INTOEXISTS)`      |
| **Materialization** | `SUBQUERY(MATERIALIZATION)` |

```sql
SELECT /*+ SUBQUERY(MATERIALIZATION) */ *
FROM employees e
WHERE e.emp_no NOT IN (
    SELECT de.emp_no
    FROM dept_emp de
    WHERE de.dept_no = 'd001'
);
```

### 9.4.2.6 BNL / NO_BNL / HASHJOIN / NO_HASHJOIN

**버전별 차이점:**

- **원래**: BNL은 Block Nested Loop Join 유도 용도
- **MySQL 8.0.20 이후**: 내부적으로 해시 조인이 도입되며, BNL 힌트의 의미가 **해시 조인 유도** 쪽으로 변경
- **HASHJOIN / NO_HASHJOIN**: 특정 버전에만 유효했고 이후 변경/무력화된 이력이 있음

```sql
-- MySQL 8.0.20 이후: 해시 조인 유도
SELECT /*+ BNL(t1, t2) */ *
FROM table1 t1
INNER JOIN table2 t2 ON t1.id = t2.id;

-- 해시 조인 비활성화
SELECT /*+ NO_BNL(t1, t2) */ *
FROM table1 t1
INNER JOIN table2 t2 ON t1.id = t2.id;
```

### 9.4.2.7 JOIN_FIXED_ORDER / JOIN_ORDER / JOIN_PREFIX / JOIN_SUFFIX

`STRAIGHT_JOIN`과 유사하게 조인 순서를 제어하는 힌트들입니다.

| 힌트                 | 설명                               |
| -------------------- | ---------------------------------- |
| **JOIN_FIXED_ORDER** | FROM에 작성된 순서를 고정          |
| **JOIN_ORDER**       | 힌트에 명시된 순서를 따르도록 유도 |
| **JOIN_PREFIX**      | 앞쪽(드라이빙) 테이블을 특정       |
| **JOIN_SUFFIX**      | 마지막(드리븐) 테이블 쪽을 특정    |

```sql
SELECT /*+ JOIN_ORDER(d, de) */ *
FROM employees e
  INNER JOIN dept_emp de ON de.emp_no = e.emp_no
  INNER JOIN departments d ON d.dept_no = de.dept_no;
```

### 9.4.2.8 MERGE / NO_MERGE

파생 테이블(derived table: FROM절 서브쿼리)을 외부 쿼리와 병합(MERGE)할지, 임시 테이블로 물리화(Materialize)할지를 옵티마이저가 선택합니다. `MERGE`/`NO_MERGE`로 이 선택을 유도합니다.

```sql
-- 파생 테이블 병합 유도
SELECT /*+ MERGE(dt) */ *
FROM (
    SELECT emp_no, COUNT(*) as cnt
    FROM dept_emp
    GROUP BY emp_no
) AS dt
WHERE dt.cnt > 1;

-- 파생 테이블 물리화 유도
SELECT /*+ NO_MERGE(dt) */ *
FROM (
    SELECT emp_no, COUNT(*) as cnt
    FROM dept_emp
    GROUP BY emp_no
) AS dt
WHERE dt.cnt > 1;
```

### 9.4.2.9 INDEX_MERGE / NO_INDEX_MERGE

MySQL은 가능하면 테이블당 인덱스 1개로 처리하려 하지만, 경우에 따라 여러 인덱스를 조합(합집합/교집합)해서 범위를 줄이는 **Index Merge**를 사용합니다.

```sql
-- Index Merge 활성화
SELECT /*+ INDEX_MERGE(employees ix_firstname, ix_lastname) */ *
FROM employees
WHERE firstname = 'Matt' OR lastname = 'Smith';

-- Index Merge 비활성화
SELECT /*+ NO_INDEX_MERGE(employees) */ *
FROM employees
WHERE firstname = 'Matt' OR lastname = 'Smith';
```

### 9.4.2.10 NO_ICP (Index Condition Pushdown 비활성화)

ICP(Index Condition Pushdown)는 대개 성능에 유리하지만, 드물게 실행 계획 추정/선택에 부정적 영향을 줄 수 있어 **비활성화 힌트(NO_ICP)**만 제공합니다.

```sql
SELECT /*+ NO_ICP(employees) */ *
FROM employees
WHERE firstname = 'Matt' AND lastname LIKE 'S%';
```

### 9.4.2.11 SKIP_SCAN / NO_SKIP_SCAN

인덱스 선행 컬럼 조건이 없어도 인덱스를 활용할 수 있게 하는 최적화가 **Skip Scan**입니다.

**특징:**

- 선행 컬럼의 유니크 값이 너무 많으면 오히려 비효율적
- 옵티마이저 판단이 틀릴 때 `NO_SKIP_SCAN`으로 방지

```sql
-- 복합 인덱스: (dept_no, emp_no)
-- Skip Scan 사용
SELECT *
FROM dept_emp
WHERE emp_no = 10001;  -- 선행 컬럼(dept_no) 조건 없음

-- Skip Scan 비활성화
SELECT /*+ NO_SKIP_SCAN(dept_emp) */ *
FROM dept_emp
WHERE emp_no = 10001;
```

### 9.4.2.12 INDEX / NO_INDEX (옵티마이저 힌트로 인덱스 힌트 대체)

옛 인덱스 힌트들을 `/*+ ... */` 형태로 대체한 것입니다.

| 인덱스 힌트                 | 옵티마이저 힌트  |
| --------------------------- | ---------------- |
| `USE INDEX`                 | `INDEX`          |
| `USE INDEX FOR GROUP BY`    | `GROUP_INDEX`    |
| `USE INDEX FOR ORDER BY`    | `ORDER_INDEX`    |
| `IGNORE INDEX`              | `NO_INDEX`       |
| `IGNORE INDEX FOR GROUP BY` | `NO_GROUP_INDEX` |
| `IGNORE INDEX FOR ORDER BY` | `NO_ORDER_INDEX` |

```sql
-- 인덱스 힌트 (구식)
EXPLAIN
SELECT *
FROM employees USE INDEX(ix_firstname)
WHERE firstname = 'Matt';

-- 옵티마이저 힌트 (권장)
EXPLAIN
SELECT /*+ INDEX(employees ix_firstname) */ *
FROM employees
WHERE firstname = 'Matt';
```

---

### 1. 힌트 사용 전 확인사항

힌트를 사용하기 전에 다음을 확인하세요:

1. **통계 정보 최신화**: `ANALYZE TABLE` 실행
2. **인덱스 존재 여부**: `SHOW INDEX FROM table_name`
3. **실행 계획 확인**: `EXPLAIN` 또는 `EXPLAIN FORMAT=JSON`
4. **성능 측정**: 힌트 적용 전후 성능 비교

### 2. 힌트 디버깅

힌트가 적용되었는지 확인:

```sql
-- 실행 계획 확인
EXPLAIN FORMAT=JSON
SELECT /*+ INDEX(employees ix_firstname) */ *
FROM employees
WHERE firstname = 'Matt';

-- 힌트 무시 여부 확인 (경고 메시지)
SHOW WARNINGS;
```
