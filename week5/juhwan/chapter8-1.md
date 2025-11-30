# MySQL 인덱스 (B-Tree 인덱스)

> **핵심 메시지**  
> 인덱스는 데이터베이스 성능의 핵심 요소로, 정렬된 구조를 유지하여 빠른 검색을 제공한다. 올바른 인덱스 설계는 옵티마이저보다 더 중요하며, 물리 모델링 단계부터 고려해야 한다.

---

## 1. 인덱스 개요

### 1.1 인덱스의 역할

- **데이터베이스 성능의 핵심 요소**: 쿼리 성능을 결정하는 가장 중요한 요소
- **정렬된 구조 유지**: 빠른 검색을 위해 키 값을 정렬된 상태로 관리
- **MySQL 8.0 기준**: 대부분의 인덱스 기능은 InnoDB 스토리지 엔진 기준으로 제공

### 1.2 인덱스의 중요성

| 관점 | 설명 |
| --- | --- |
| **설계 시점** | 물리 모델링 단계부터 인덱스를 고려해야 함 |
| **성능 영향** | 잘못된 인덱스 설계는 심각한 성능 저하 초래 |
| **옵티마이저와의 관계** | 옵티마이저가 아무리 좋아도 인덱스 설계가 더 중요 |

### 1.3 인덱스의 장단점

#### 장점
- ✅ **빠른 검색**: O(log n) 시간 복잡도로 데이터 검색
- ✅ **정렬 최적화**: ORDER BY 절의 정렬 비용 제거
- ✅ **조인 최적화**: 조인 연산 시 성능 향상
- ✅ **유일성 보장**: UNIQUE 인덱스로 데이터 무결성 보장

#### 단점
- ❌ **저장 공간**: 인덱스는 추가 저장 공간 필요 (테이블 크기의 10~20%)
- ❌ **쓰기 성능 저하**: INSERT, UPDATE, DELETE 시 인덱스도 함께 갱신 필요
- ❌ **유지보수 비용**: 인덱스가 많을수록 유지보수 비용 증가

---

## 2. B-Tree 인덱스 구조 및 특성

### 2.1 B-Tree 구조

B-Tree 인덱스는 계층적 구조를 가진 균형 트리입니다.

```
┌─────────────────┐
│   루트 페이지    │  ← 최상위 노드
│  (Root Page)    │
└────────┬────────┘
         │
    ┌────┴────┐
    │        │
┌───▼───┐ ┌──▼────┐
│ 브랜치 │ │ 브랜치│  ← 중간 노드
│ 페이지 │ │ 페이지│
└───┬───┘ └──┬────┘
    │        │
┌───▼───┐ ┌──▼────┐
│ 리프  │ │ 리프  │  ← 최하위 노드
│ 페이지│ │ 페이지│  (실제 키 값 저장)
└───────┘ └───────┘
```

#### 구조 특징
- **루트 페이지**: 최상위 노드, 브랜치 페이지 또는 리프 페이지를 가리킴
- **브랜치 페이지**: 중간 노드, 하위 페이지를 가리키는 포인터 포함
- **리프 페이지**: 최하위 노드, **실제 키 값과 데이터 포인터 저장**
- **균형 트리**: 모든 리프 노드까지의 깊이가 동일 (검색 성능 일정)

### 2.2 B-Tree 인덱스의 주요 특징

| 특징 | 설명 | 영향 |
| --- | --- | --- |
| **정렬된 상태 유지** | 키 값이 항상 정렬된 상태로 저장 | 범위 검색에 유리 |
| **범위 검색 최적화** | `>`, `<`, `BETWEEN` 등 범위 조건에 효율적 | WHERE 절 최적화 |
| **변경 작업 비용** | INSERT/DELETE/UPDATE 시 인덱스 재구성 필요 | 쓰기 성능 저하 |
| **UPDATE 처리** | 기존 키 삭제 + 새 키 삽입 (완전 재배치) | UPDATE 비용 높음 |

### 2.3 키 삭제/변경 처리 방식

#### 삭제 (DELETE)
- **처리 방식**: 리프 노드에서 **마킹만 수행** (지연 처리 가능)
- **특징**: 실제 물리적 삭제는 나중에 수행 (Purge 작업)

#### 변경 (UPDATE)
- **처리 방식**: 기존 키 삭제 → 새 키 삽입 (완전 재배치)
- **비용**: 인덱스 키가 변경되면 두 번의 작업 필요

#### InnoDB의 최적화
- **체인지 버퍼(Change Buffer)**: 변경 작업을 버퍼에 지연 반영하여 성능 향상
- **지연 쓰기**: 인덱스 변경을 메모리에 먼저 기록 후 배치로 디스크에 반영

---

## 3. B-Tree 인덱스 스캔 방식

MySQL은 쿼리 조건에 따라 다양한 방식으로 인덱스를 스캔합니다.

### 3.1 인덱스 레인지 스캔 (Index Range Scan)

**가장 이상적인 인덱스 활용 방식**

#### 사용 조건
- `=` (등호 조건)
- `>`, `<`, `>=`, `<=` (범위 조건)
- `BETWEEN` (범위 조건)
- `IN` (여러 값 비교)

#### 예시
```sql
-- 인덱스: (user_id, created_at)
SELECT * FROM orders 
WHERE user_id = 100 
  AND created_at BETWEEN '2024-01-01' AND '2024-01-31';
-- → 인덱스 레인지 스캔 사용
```

#### 특징
- ✅ 인덱스의 특정 범위만 스캔
- ✅ 가장 효율적인 스캔 방식
- ✅ 필요한 데이터만 빠르게 찾음

---

### 3.2 인덱스 풀 스캔 (Index Full Scan)

**인덱스 전체를 처음부터 끝까지 스캔**

#### 사용 조건
- 인덱스의 **첫 번째 칼럼 조건이 없는 경우**
- 예: 인덱스 `(A, B, C)`인데 `WHERE B = 10` 조건만 있는 경우

#### 예시
```sql
-- 인덱스: (gender, birth_date)
SELECT * FROM users 
WHERE birth_date = '1990-01-01';
-- → gender 조건이 없으므로 인덱스 풀 스캔
```

#### 특징
- ⚠️ 인덱스 전체를 스캔하지만 **테이블 풀 스캔보다는 빠름**
- ⚠️ 인덱스는 테이블보다 크기가 훨씬 작기 때문
- ⚠️ 리프 페이지를 순차적으로 모두 읽음

#### 성능 비교
```
테이블 풀 스캔 > 인덱스 풀 스캔 > 인덱스 레인지 스캔
(느림)          (보통)          (빠름)
```

---

### 3.3 인덱스 스킵 스캔 (Index Skip Scan) - MySQL 8.0+

**선행 칼럼 조건이 없어도 후행 칼럼만으로 검색 가능**

#### 배경
- **기존 방식**: 인덱스 `(gender, birth_date)`에서 `gender` 조건이 없으면 인덱스 사용 불가
- **8.0 개선**: `gender` 없이 `birth_date`만으로도 인덱스 검색 가능

#### 작동 방식
1. 선행 칼럼(`gender`)의 **유니크한 값들을 추출**
2. 각 유니크 값마다 후행 칼럼(`birth_date`) 조건을 만족하는 범위를 탐색
3. 결과를 병합

#### 예시
```sql
-- 인덱스: (gender, birth_date)
SELECT * FROM users 
WHERE birth_date = '1990-01-01';
-- → gender 조건 없지만 스킵 스캔으로 인덱스 사용 가능

-- 내부 동작:
-- 1. gender의 유니크 값 추출: 'M', 'F'
-- 2. (gender='M', birth_date='1990-01-01') 범위 스캔
-- 3. (gender='F', birth_date='1990-01-01') 범위 스캔
-- 4. 결과 병합
```

#### 효율성
- **선행 칼럼의 distinct 값이 적을수록 효율적** (예: gender는 2~3개)
- **선행 칼럼의 distinct 값이 많을수록 비효율적** (거의 풀 스캔과 유사)

#### 설정 및 제어
```sql
-- 설정 확인
SHOW VARIABLES LIKE 'optimizer_switch';
-- skip_scan=on/off

-- 힌트로 비활성화
SELECT /*+ NO_SKIP_SCAN(users idx_gender_birth) */ * 
FROM users 
WHERE birth_date = '1990-01-01';
```

---

## 4. 다중 칼럼 인덱스 (Multi-column Index)

### 4.1 Leftmost Prefix 원칙

**인덱스는 왼쪽부터 순서대로 사용됨**

#### 인덱스 구성 예시: `(A, B, C)`

| 사용 가능한 조합 | 사용 불가능한 조합 |
| --- | --- |
| ✅ `A` | ❌ `B` |
| ✅ `A, B` | ❌ `B, C` |
| ✅ `A, B, C` | ❌ `C` |
| ✅ `A, C` (부분 사용) | ❌ `B`만 단독 사용 |

#### 예시
```sql
-- 인덱스: (user_id, status, created_at)

-- ✅ 사용 가능
WHERE user_id = 100
WHERE user_id = 100 AND status = 'active'
WHERE user_id = 100 AND status = 'active' AND created_at > '2024-01-01'

-- ❌ 사용 불가능 (풀 스캔 또는 스킵 스캔)
WHERE status = 'active'
WHERE status = 'active' AND created_at > '2024-01-01'
```

### 4.2 다중 칼럼 인덱스 설계 가이드라인

#### 1. 선택도(Cardinality)가 높은 칼럼을 앞에
- **선택도**: 고유한 값의 비율이 높을수록 선택도가 높음
- **원칙**: 선택도가 높은 칼럼을 선행 칼럼으로 배치

```sql
-- 예시: users 테이블
-- user_id: 선택도 매우 높음 (거의 유니크)
-- gender: 선택도 낮음 (M/F 두 개)
-- birth_date: 선택도 중간

-- ✅ 좋은 인덱스
CREATE INDEX idx_user_birth ON users(user_id, birth_date);

-- ❌ 나쁜 인덱스
CREATE INDEX idx_gender_user ON users(gender, user_id);
```

#### 2. 조건절 빈도 고려
- 자주 사용되는 조건 칼럼을 선행 칼럼으로

#### 3. 정렬 기준 고려
- `ORDER BY`에 자주 사용되는 칼럼을 인덱스에 포함

#### 4. 등호 조건을 범위 조건보다 앞에
```sql
-- ✅ 좋은 순서
WHERE user_id = 100 AND created_at > '2024-01-01'
-- 인덱스: (user_id, created_at)

-- ⚠️ 나쁜 순서
WHERE created_at > '2024-01-01' AND user_id = 100
-- 인덱스: (created_at, user_id) → user_id 조건이 범위 스캔 후 적용
```

---

## 5. B-Tree 인덱스의 정렬 및 스캔 방향

### 5.1 인덱스와 정렬

**인덱스는 항상 정렬된 상태를 유지**

#### ORDER BY 최적화
- 인덱스가 정렬되어 있으므로 **별도의 정렬 작업 불필요**
- 인덱스를 따라 스캔하면 자동으로 정렬된 결과 제공

#### 예시
```sql
-- 인덱스: (user_id, created_at)

-- ✅ 정렬 최적화됨
SELECT * FROM orders 
WHERE user_id = 100 
ORDER BY created_at;
-- → 인덱스가 이미 정렬되어 있어 추가 정렬 불필요

-- ❌ 정렬 최적화 안 됨
SELECT * FROM orders 
WHERE user_id = 100 
ORDER BY created_at DESC, amount;
-- → amount는 인덱스에 없어 추가 정렬 필요
```

### 5.2 정방향 / 역방향 스캔

#### MySQL 8.0 개선
- **이전 버전**: 역순 스캔이 비효율적
- **8.0**: 인덱스 역순 스캔도 효율적으로 처리 가능

#### 예시
```sql
-- 인덱스: (created_at)

-- 정방향 스캔
SELECT * FROM orders 
ORDER BY created_at ASC;
-- → 인덱스 앞에서부터 스캔

-- 역방향 스캔 (8.0에서 효율적)
SELECT * FROM orders 
ORDER BY created_at DESC;
-- → 인덱스 뒤에서부터 역순 스캔
```

#### 주의사항
- 인덱스 구성 및 정렬 방식에 따라 실행 계획이 달라질 수 있음
- `EXPLAIN`으로 실제 실행 계획 확인 필요

---

## 6. 인덱스 선택도 (Cardinality)

### 6.1 선택도란?

**칼럼의 고유한 값의 비율**

- **높은 선택도**: 거의 모든 값이 고유함 (예: user_id, email)
- **낮은 선택도**: 중복된 값이 많음 (예: gender, status)

### 6.2 선택도와 인덱스 효율성

| 선택도 | 예시 | 인덱스 효율성 |
| --- | --- | --- |
| **매우 높음** | user_id, email | 매우 효율적 |
| **높음** | birth_date, phone | 효율적 |
| **중간** | city, category | 보통 |
| **낮음** | gender, status | 비효율적 (풀 스캔 고려) |

### 6.3 선택도 확인 방법

```sql
-- 칼럼의 선택도 확인
SELECT 
    COUNT(DISTINCT column_name) AS distinct_count,
    COUNT(*) AS total_count,
    COUNT(DISTINCT column_name) / COUNT(*) AS selectivity
FROM table_name;

-- 인덱스의 카디널리티 확인
SHOW INDEX FROM table_name;
```

---

## 7. 커버링 인덱스 (Covering Index)

### 7.1 커버링 인덱스란?

**쿼리에 필요한 모든 데이터가 인덱스에 포함되어 있어 테이블 접근이 불필요한 경우**

### 7.2 커버링 인덱스의 장점

- ✅ **테이블 접근 불필요**: 인덱스만 읽으면 되므로 매우 빠름
- ✅ **I/O 감소**: 디스크 읽기 횟수 최소화
- ✅ **성능 향상**: 특히 대용량 테이블에서 효과적

### 7.3 예시

```sql
-- 테이블: users (id, name, email, created_at)
-- 인덱스: (email, name)

-- ✅ 커버링 인덱스 사용
SELECT name, email 
FROM users 
WHERE email = 'user@example.com';
-- → 인덱스에 name, email 모두 포함되어 테이블 접근 불필요

-- ❌ 커버링 인덱스 미사용
SELECT id, name, email, created_at 
FROM users 
WHERE email = 'user@example.com';
-- → created_at이 인덱스에 없어 테이블 접근 필요
```

### 7.4 실행 계획 확인

```sql
EXPLAIN SELECT name, email 
FROM users 
WHERE email = 'user@example.com';

-- Extra 컬럼에 "Using index" 표시되면 커버링 인덱스 사용
```

---

## 8. 인덱스 설계 실무 가이드라인

### 8.1 인덱스 생성 원칙

1. **WHERE 절에 자주 사용되는 칼럼**
2. **JOIN 조건에 사용되는 칼럼**
3. **ORDER BY에 자주 사용되는 칼럼**
4. **선택도가 높은 칼럼 우선**

### 8.2 인덱스 생성 시 주의사항

#### ❌ 인덱스를 만들지 말아야 할 경우
- 선택도가 매우 낮은 칼럼 (예: gender, boolean 플래그)
- 자주 변경되는 칼럼 (UPDATE 비용 증가)
- 작은 테이블 (풀 스캔이 더 빠를 수 있음)

#### ⚠️ 인덱스 과다 생성 주의
- 인덱스가 많을수록 쓰기 성능 저하
- 저장 공간 증가
- 옵티마이저의 선택지가 많아져 실행 계획 수립 시간 증가

### 8.3 인덱스 모니터링

```sql
-- 인덱스 사용 통계 확인
SELECT 
    object_schema,
    object_name,
    index_name,
    count_read,
    sum_timer_read,
    count_write,
    sum_timer_write
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'your_database'
ORDER BY sum_timer_read DESC;

-- 사용되지 않는 인덱스 찾기
-- count_read가 0이거나 매우 낮은 인덱스는 제거 고려
```

---

## 9. 실행 계획 분석 (EXPLAIN)

### 9.1 EXPLAIN 사용법

```sql
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
```

### 9.2 주요 컬럼 의미

| 컬럼 | 의미 | 중요도 |
| --- | --- | --- |
| **type** | 조인 타입 (접근 방식) | ⭐⭐⭐ |
| **key** | 사용된 인덱스 | ⭐⭐⭐ |
| **rows** | 예상 검색 행 수 | ⭐⭐ |
| **Extra** | 추가 정보 (Using index, Using filesort 등) | ⭐⭐ |

### 9.3 type 컬럼 값

| 값 | 설명 | 성능 |
| --- | --- | --- |
| **const** | PRIMARY KEY나 UNIQUE 인덱스로 단일 행 조회 | 최고 |
| **ref** | 인덱스로 단일 행 조회 | 매우 좋음 |
| **range** | 인덱스 레인지 스캔 | 좋음 |
| **index** | 인덱스 풀 스캔 | 보통 |
| **ALL** | 테이블 풀 스캔 | 나쁨 |

---

## 발표용 요약

### 핵심 개념
1. **인덱스는 정렬된 구조**: 빠른 검색과 정렬 최적화 제공
2. **Leftmost Prefix**: 다중 칼럼 인덱스는 왼쪽부터 순서대로 사용
3. **스캔 방식**: 레인지 스캔 > 풀 스캔 > 테이블 풀 스캔

### 실무 팁
- ✅ 선택도가 높은 칼럼을 인덱스 앞에 배치
- ✅ WHERE, JOIN, ORDER BY에 자주 사용되는 칼럼에 인덱스 생성
- ✅ 커버링 인덱스 활용으로 성능 극대화
- ❌ 선택도가 낮거나 자주 변경되는 칼럼은 인덱스 지양
- ❌ 인덱스 과다 생성 주의

---

## 토론 질문 (Discussion Prompts)

1. 우리 애플리케이션에서 가장 자주 사용되는 쿼리는 무엇이며, 적절한 인덱스가 있는가?
2. 다중 칼럼 인덱스에서 칼럼 순서를 결정할 때 어떤 기준을 사용하는가?
3. 인덱스 스킵 스캔이 실제로 사용되는 쿼리가 있는가? 성능은 어떤가?
4. 사용되지 않는 인덱스를 찾아 제거한 경험이 있는가?
