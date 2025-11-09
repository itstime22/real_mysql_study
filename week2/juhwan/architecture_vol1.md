# MySQL 아키텍처 및 InnoDB 스토리지 엔진

---

## 📌 Part 1: MySQL 엔진 아키텍처

---

### ✅ 4.1.1 MySQL의 전체 구조

#### 🏗️ 계층 구조

MySQL 서버는 **두 계층**으로 구성되어 있습니다:

```
┌─────────────────────────────────────┐
│   MySQL 엔진 (SQL Layer)            │
│   - SQL 파서                        │
│   - 전처리기                        │
│   - 옵티마이저                      │
│   - SQL 실행기                      │
│   - 캐시 & 버퍼 관리                │
└─────────────────────────────────────┘
           ↓ API 호출
┌─────────────────────────────────────┐
│   스토리지 엔진 (Storage Engine)    │
│   - InnoDB (기본)                   │
│   - MyISAM                          │
│   - MEMORY                          │
└─────────────────────────────────────┘
```

#### 📋 역할 분담

| 계층 | 역할 | 담당 기능 |
|------|------|----------|
| **MySQL 엔진** | SQL 처리 로직 | SQL 파싱, 최적화, 실행 계획 수립 |
| **스토리지 엔진** | 데이터 저장/읽기 | 실제 디스크/메모리 I/O 처리 |

#### 💡 MySQL의 핵심 특징

- **플러그인 기반 스토리지 엔진 구조**
  - 엔진을 모듈처럼 교체 가능한 유연한 아키텍처
  - 테이블별로 다른 스토리지 엔진 사용 가능
  - 엔진마다 트랜잭션 지원, 인덱스 구조, 잠금 방식이 다름

**예시:**
```sql
-- 테이블 생성 시 스토리지 엔진 지정
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
) ENGINE=InnoDB;  -- InnoDB 엔진 사용

CREATE TABLE logs (
    id INT PRIMARY KEY,
    message TEXT
) ENGINE=MyISAM;  -- MyISAM 엔진 사용
```

---

### ✅ 4.1.2 MySQL 스레딩 구조

#### 🔄 스레드 모델

**1. 포그라운드 스레드 (Foreground Thread)**
- **1-커넥션-1-스레드 모델**
  - 클라이언트 연결 시 전용 스레드 생성
  - 각 커넥션마다 독립적인 스레드 할당
  - 쿼리 처리 담당

**특징:**
- 장점: 간단하고 이해하기 쉬운 구조
- 단점: 대규모 트래픽 시 스레드 수 증가 → 컨텍스트 스위칭 비용 증가

**2. 백그라운드 스레드 (Background Thread)**
- InnoDB 내부에서 사용되는 시스템 스레드
- 주요 백그라운드 스레드:
  - **I/O 스레드**: 디스크 읽기/쓰기 작업
  - **버퍼 풀 플러시 스레드**: Dirty Page를 디스크에 기록
  - **페이지 정리 스레드**: 페이지 정리 및 최적화
  - **로그 스레드**: Redo Log 기록

**✨ 유용한 팁**
```sql
-- 현재 활성 커넥션 수 확인
SHOW STATUS LIKE 'Threads_connected';

-- 최대 동시 커넥션 수 확인
SHOW VARIABLES LIKE 'max_connections';
```

---

### ✅ 4.1.3 메모리 할당 및 사용 구조

#### 💾 메모리 영역 분류

MySQL 메모리는 크게 **2가지**로 구분됩니다:

**1. 글로벌 메모리 영역 (Global Memory)**
- 모든 스레드가 공유하는 메모리
- 서버 시작 시 한 번만 할당
- 주요 예시:
  - **InnoDB Buffer Pool**: 가장 중요한 캐시 영역
  - **Key Buffer**: MyISAM 인덱스 캐시
  - ~~Query Cache~~: MySQL 8.0에서 제거됨

**2. 세션 메모리 영역 (Session Memory)**
- 각 클라이언트 연결(세션)마다 독립적으로 할당
- 커넥션 수가 많을수록 총 메모리 사용량 증가
- 주요 예시:
  - **Sort Buffer**: ORDER BY, GROUP BY 정렬 작업
  - **Join Buffer**: JOIN 연산 시 사용
  - **Read Buffer**: 순차 스캔 시 사용
  - **Temporary Table**: 임시 테이블 생성

#### ⚠️ 주의사항

1. **커넥션 수 관리**
   - 커넥션 풀(Connection Pool) 사용 권장
   - 불필요한 커넥션은 즉시 종료
   - `max_connections` 설정을 적절히 조정

2. **버퍼 크기 조정**
```sql
-- 세션별 버퍼 크기 확인
SHOW VARIABLES LIKE '%buffer%';

-- 주요 설정
SET SORT_BUFFER_SIZE = 256 * 1024;  -- Sort Buffer 크기
SET JOIN_BUFFER_SIZE = 256 * 1024;  -- Join Buffer 크기
```

3. **메모리 모니터링**
```sql
-- 현재 메모리 사용량 확인
SHOW STATUS LIKE 'Memory%';
```

---

### ✅ 4.1.4 플러그인 스토리지 엔진 모델

#### 🔌 플러그인 시스템

MySQL의 강력한 구조적 특징 중 하나는 **플러그인 기반 아키텍처**입니다.

**주요 기능:**
- 스토리지 엔진을 모듈처럼 교체 가능
- 서버 재시작 없이 플러그인 설치/제거 가능
- 다양한 플러그인 타입 지원

**플러그인 종류:**
1. **스토리지 엔진**: InnoDB, MyISAM, MEMORY 등
2. **인증 플러그인**: 다양한 인증 방식 지원
3. **전문 검색용 파서**: Full-text 검색 기능
4. **Audit 플러그인**: 보안 감사 로그

**확인 방법:**
```sql
-- 설치된 플러그인 조회
SHOW PLUGINS;

-- 특정 플러그인 상태 확인
SHOW PLUGINS LIKE 'InnoDB';
```

#### 📊 주요 스토리지 엔진 비교

| 엔진 | 트랜잭션 | 외래키 | 잠금 단위 | 사용 시나리오 |
|------|----------|--------|-----------|--------------|
| **InnoDB** | ✅ 지원 | ✅ 지원 | Row-level | 일반적인 OLTP |
| **MyISAM** | ❌ 미지원 | ❌ 미지원 | Table-level | 읽기 위주, 로그 |
| **MEMORY** | ❌ 미지원 | ❌ 미지원 | Table-level | 임시 데이터, 캐시 |

---

### ✅ 4.1.5 컴포넌트(Component) - MySQL 8.0 신규 기능

#### 🆕 컴포넌트 vs 플러그인

MySQL 8.0에서 도입된 **컴포넌트**는 플러그인의 한계를 해결하기 위해 개발되었습니다.

**플러그인의 문제점:**
- 플러그인 간 통신 불가
- MySQL 서버 내부 구조에 과도하게 의존
- 초기화 및 의존성 관리 어려움
- 보안 취약점 가능성

**컴포넌트의 장점:**
- ✅ 컴포넌트 간 상호 의존성 관리 가능
- ✅ 더 안정적인 인터페이스 제공
- ✅ 보안 개선 (격리된 환경)
- ✅ 초기화 순서 관리 용이

**사용 예시:**
```sql
-- 컴포넌트 설치
INSTALL COMPONENT 'file://component_validate_password';

-- 컴포넌트 확인
SELECT * FROM mysql.component;
```

---

### ✅ 4.1.6 쿼리 실행 구조

#### 🔄 SQL 실행 흐름

쿼리 실행은 다음 단계를 거칩니다:

```
1. Parser (파서)
   ↓
2. Preprocessor (전처리기)
   ↓
3. Optimizer (옵티마이저)
   ↓
4. Executor (실행기)
   ↓
5. Storage Engine (스토리지 엔진)
```

#### 📝 각 단계 상세 설명

**1. Parser (파서)**
- SQL 문법 분석 및 검증
- 토큰 분해 (Lexical Analysis)
- AST(Abstract Syntax Tree) 생성
- 문법 오류 검출

**2. Preprocessor (전처리기)**
- 객체 존재 여부 확인 (테이블, 컬럼)
- 권한 검사 (사용자 권한 확인)
- 구조적 오류 검출
- 매크로 확장

**3. Optimizer (옵티마이저)**
- **실행 계획(Execution Plan) 수립**
- 비용 기반 최적화 (Cost-based Optimization)
- 인덱스 선택
- 조인 순서 결정
- 서브쿼리 최적화

**실행 계획 확인:**
```sql
-- 실행 계획 확인
EXPLAIN SELECT * FROM users WHERE id = 1;

-- 상세 실행 계획 (MySQL 8.0)
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE id = 1;
```

**4. Executor (실행기)**
- 옵티마이저가 만든 계획대로 실제 실행
- 스토리지 엔진 API 호출
- 데이터 읽기/쓰기 요청

**5. Storage Engine (스토리지 엔진)**
- 실제 데이터 I/O 처리
- 디스크/메모리에서 데이터 읽기/쓰기

#### 💡 핵심 포인트

> **MySQL 엔진은 SQL 처리 로직을 담당하고, 실제 데이터 관리는 스토리지 엔진이 담당합니다.**

이러한 역할 분담으로 인해:
- SQL 처리 로직과 데이터 저장 로직의 독립성 확보
- 다양한 스토리지 엔진 교체 가능
- 엔진별 최적화 전략 적용 가능

---

### ✅ 4.1.7 복제 (Replication)

#### 🔄 MySQL 복제 구조

MySQL 복제는 **마스터-슬레이브 구조**로 데이터를 동기화합니다.

**핵심 구성 요소:**
1. **Binary Log (바이너리 로그)**
   - 마스터 서버의 모든 변경 사항 기록
   - DML(INSERT, UPDATE, DELETE) 및 DDL 기록

2. **Replication I/O Thread**
   - 슬레이브에서 마스터의 Binary Log 읽기
   - Relay Log에 저장

3. **Replication SQL Thread**
   - Relay Log의 변경 사항을 슬레이브에 적용

**복제 구조:**
```
마스터 서버              슬레이브 서버
    │                       │
    │  Binary Log           │
    │  (변경 이력)          │
    │       │               │
    │       ├──────────────>│  I/O Thread
    │       │               │  (읽기)
    │       │               │
    │       │          Relay Log
    │       │               │
    │       │               │  SQL Thread
    │       │               │  (적용)
    │       │               │
```

**복제 종류:**
1. **비동기 복제 (Asynchronous Replication)** - 기본
   - 마스터가 슬레이브 응답을 기다리지 않음
   - 성능 우수, 데이터 일관성 보장 약함

2. **준동기 복제 (Semi-synchronous Replication)** - 옵션
   - 최소 1개 슬레이브에 전송 확인 후 커밋
   - 성능과 일관성의 균형

3. **그룹 복제 (Group Replication)** - MySQL 8.0
   - 다중 마스터 복제
   - 자동 장애 감지 및 복구

**복제 설정 확인:**
```sql
-- 복제 상태 확인
SHOW SLAVE STATUS\G

-- Binary Log 상태 확인
SHOW BINARY LOGS;
SHOW BINLOG EVENTS;
```

---

### ✅ 4.1.8 쿼리 캐시 (Query Cache)

#### ⚠️ MySQL 8.0에서 제거됨

**제거 이유:**
- 동시성 문제: 락 경합으로 인한 성능 저하
- 캐시 무효화 오버헤드
- 메모리 사용 비효율
- 대부분의 워크로드에서 효과 미미

**대안:**
- 애플리케이션 레벨 캐싱 (Redis, Memcached)
- InnoDB Buffer Pool 최적화
- 쿼리 최적화

---

### ✅ 4.1.9 스레드 풀 (Thread Pool)

#### 🎯 스레드 관리 최적화

스레드 풀은 **고성능 서버**에서 매우 유용한 기능입니다.

**작동 원리:**
- 많은 커넥션이 들어와도 스레드 수가 폭증하지 않음
- CPU 코어 수 기반으로 스레드 풀 생성
- 작업을 스레드 풀에 배분하여 처리

**장점:**
- 컨텍스트 스위칭 비용 감소
- 메모리 사용량 감소
- 시스템 안정성 향상

**사용 시나리오:**
- 동시 연결 수가 많은 환경 (수천 개 이상)
- 짧은 쿼리가 많은 OLTP 워크로드
- CPU 코어가 충분한 환경

**설정:**
```sql
-- 스레드 풀 플러그인 설치 (별도 설치 필요)
INSTALL PLUGIN thread_pool SONAME 'thread_pool.so';

-- 스레드 풀 크기 설정
SET GLOBAL thread_pool_size = 16;  -- CPU 코어 수와 유사하게 설정
```

---

### ✅ 4.1.10 트랜잭션 지원 메타데이터

#### 📊 데이터 딕셔너리

MySQL 8.0부터 **데이터 딕셔너리 메타데이터가 InnoDB 테이블로 통합**되었습니다.

**변경 사항:**
- 이전: `.frm` 파일에 메타데이터 저장 (파일 시스템)
- 현재: InnoDB 테이블에 메타데이터 저장 (트랜잭션 지원)

**장점:**
- 트랜잭션 지원: 메타데이터 변경도 원자성 보장
- 일관성 보장: 크래시 시에도 일관된 상태 유지
- 성능 향상: 파일 시스템 I/O 감소

**확인 방법:**
```sql
-- 시스템 테이블스페이스 확인
SHOW VARIABLES LIKE 'innodb_data_file_path';

-- 데이터 딕셔너리 정보 (MySQL 8.0)
SELECT * FROM information_schema.INNODB_TABLES;
```

---

## 📌 Part 2: InnoDB 스토리지 엔진 아키텍처

---

### ✅ 4.2.1 프라이머리 키에 의한 클러스터링

#### 🎯 InnoDB의 핵심 원리

InnoDB는 **프라이머리 키(PRIMARY KEY) 순서대로 데이터를 물리적으로 저장**합니다.

**클러스터링 인덱스:**
- 프라이머리 키가 클러스터링 인덱스
- 데이터 자체가 인덱스 구조 (B-Tree)
- 프라이머리 키 순서 = 물리적 저장 순서

**보조 인덱스 구조:**
- 모든 보조 인덱스(Secondary Index)는 **프라이머리 키 값을 포함**
- 보조 인덱스 → 프라이머리 키 → 실제 데이터로 접근

**데이터 접근 흐름:**
```
보조 인덱스 검색
    ↓
프라이머리 키 찾기
    ↓
클러스터링 인덱스로 실제 데이터 접근
```

**예시:**
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,        -- 클러스터링 인덱스
    email VARCHAR(100),
    name VARCHAR(100),
    INDEX idx_email (email)    -- 보조 인덱스 (PK 포함)
);

-- email로 검색 시:
-- 1. idx_email 인덱스에서 PK(id) 찾기
-- 2. PK로 클러스터링 인덱스에서 실제 데이터 읽기
```

**장점:**
- ✅ **범위 검색 성능 우수**: BETWEEN, ORDER BY PK가 매우 빠름
- ✅ **PK 기반 검색이 가장 빠름**: 한 번의 인덱스 탐색으로 데이터 접근
- ✅ **페이지 접근 효율**: 연속된 데이터는 같은 페이지에 저장

**✨ 유용한 팁**
- 프라이머리 키는 반드시 설정 (AUTO_INCREMENT 권장)
- 프라이머리 키 기반 범위 검색 활용
- 보조 인덱스는 필요한 것만 생성 (인덱스 유지 비용 고려)

---

### ✅ 4.2.2 외래 키(Foreign Key) 지원

#### 🔗 참조 무결성 보장

InnoDB는 **외래 키 제약 조건**을 지원하여 참조 무결성을 보장합니다.

**주요 기능:**
- DELETE/UPDATE 시 제약 검사
- 부모 테이블 변경 시 자식 테이블 동작 정의
- 참조 무결성 자동 검증

**외래 키 동작:**
```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE      -- 부모 삭제 시 자식도 삭제
        ON UPDATE CASCADE      -- 부모 업데이트 시 자식도 업데이트
);
```

**주의사항:**
- ⚠️ **외래 키 컬럼에 인덱스 필수**: 성능을 위해 반드시 인덱스 필요
- ⚠️ **데드락 위험**: 외래 키 검사로 인한 잠금 증가
- ⚠️ **성능 오버헤드**: 대용량 테이블에서 성능 저하 가능

**권장사항:**
- 애플리케이션 레벨에서 참조 무결성 관리 고려
- 외래 키 사용 시 인덱스 반드시 생성
- 필요시 외래 키 비활성화 고려 (`SET FOREIGN_KEY_CHECKS = 0`)

---

### ✅ 4.2.3 MVCC (Multi-Version Concurrency Control)

#### 🔄 다중 버전 동시성 제어

MVCC는 **잠금 없이 일관된 읽기**를 가능하게 하는 InnoDB의 핵심 메커니즘입니다.

**MVCC 특징:**
- Undo Log 기반 읽기
- 트랜잭션 격리 수준 지원 (REPEATABLE READ)
- 읽기 작업이 쓰기 작업을 블로킹하지 않음
- 동시성 향상

**주요 구성 요소:**
1. **Undo Log**
   - 이전 버전의 데이터 저장
   - 롤백 및 일관된 읽기에 사용

2. **Read View**
   - 트랜잭션 시점의 스냅샷 생성
   - 어떤 버전의 데이터를 볼지 결정

3. **Rollback Segments**
   - Undo Log 저장 공간
   - 여러 세그먼트로 분리되어 동시성 향상

**작동 원리:**
```
트랜잭션 A (읽기)
    ↓
Read View 생성 (시점 T1)
    ↓
데이터 읽기 (Undo Log에서 이전 버전 조회)
    ↓
트랜잭션 B (쓰기) - 블로킹 없음
    ↓
새 버전 생성 (Undo Log에 이전 버전 저장)
```

**격리 수준:**
- **READ UNCOMMITTED**: Dirty Read 허용
- **READ COMMITTED**: 커밋된 데이터만 읽기
- **REPEATABLE READ** (기본): 트랜잭션 내 일관된 읽기
- **SERIALIZABLE**: 완전 격리

**확인 방법:**
```sql
-- 현재 격리 수준 확인
SELECT @@transaction_isolation;

-- 격리 수준 변경
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

---

### ✅ 4.2.4 Non-Locking Consistent Read (잠금 없는 읽기)

#### 🚀 동시성 향상 메커니즘

InnoDB는 **SELECT 쿼리가 다른 트랜잭션을 블로킹하지 않습니다**.

**작동 방식:**
- SELECT는 Undo Log 기반 Snapshot 읽기
- 쓰기 작업과 독립적으로 실행
- 대량 동시 트래픽 환경에서 매우 중요

**예시:**
```sql
-- 트랜잭션 A (읽기)
START TRANSACTION;
SELECT * FROM users WHERE id = 1;  -- 블로킹 없음
-- 다른 트랜잭션의 쓰기 작업과 동시 실행 가능

-- 트랜잭션 B (쓰기)
START TRANSACTION;
UPDATE users SET name = 'New Name' WHERE id = 1;
COMMIT;  -- 트랜잭션 A는 여전히 이전 데이터 읽기
```

**주의사항:**
- 일부 SELECT는 잠금을 사용할 수 있음:
  - `SELECT ... FOR UPDATE` (배타적 잠금)
  - `SELECT ... LOCK IN SHARE MODE` (공유 잠금)

**✨ 유용한 정보**
- 읽기 위주 워크로드에 최적
- 리포트 쿼리와 트랜잭션 쿼리 동시 실행 가능
- 대규모 동시 접속 환경에 적합

---

### ✅ 4.2.5 자동 데드락 감지

#### 🔍 데드락 처리

InnoDB는 **데드락을 자동으로 감지하고 처리**합니다.

**데드락 감지:**
- 주기적으로 데드락 검사
- 데드락 발견 시 자동 롤백
- 더 적은 비용의 트랜잭션을 롤백 (작은 트랜잭션 우선)

**데드락 예시:**
```
트랜잭션 A: Row 1 잠금 → Row 2 잠금 대기
트랜잭션 B: Row 2 잠금 → Row 1 잠금 대기
→ 데드락 발생!
```

**처리 방법:**
```sql
-- 데드락 발생 시 에러 발생
ERROR 1213 (40001): Deadlock found when trying to get lock

-- 애플리케이션에서 재시도 로직 구현 필요
```

**데드락 모니터링:**
```sql
-- 데드락 발생 횟수 확인
SHOW STATUS LIKE 'Innodb_deadlocks';

-- 최근 데드락 정보 (MySQL 8.0)
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
```

**예방 방법:**
- 트랜잭션 시간 최소화
- 동일한 순서로 잠금 획득
- 인덱스 사용으로 잠금 범위 최소화
- 데드락 발생 시 재시도 로직 구현

---

### ✅ 4.2.6 장애 복구(Crash Recovery)

#### 💪 안정성 보장

InnoDB는 **서버 장애 시에도 데이터 무결성을 보장**합니다.

**복구 메커니즘:**
1. **Redo Log**
   - 커밋된 트랜잭션의 변경 사항 기록
   - 장애 시 Redo Log를 재실행하여 복구

2. **DoubleWrite Buffer**
   - 부분 페이지 쓰기(Partial Page Write) 방지
   - 디스크 쓰기 중 장애 발생 시 복구 지원

3. **Checkpoint**
   - 주기적으로 버퍼 풀의 변경 사항을 디스크에 기록
   - 복구 시간 단축

**복구 과정:**
```
서버 재시작
    ↓
Redo Log 스캔
    ↓
커밋된 트랜잭션 재실행
    ↓
Undo Log 기반 미커밋 트랜잭션 롤백
    ↓
복구 완료
```

**설정:**
```sql
-- Redo Log 크기 확인
SHOW VARIABLES LIKE 'innodb_log_file_size';
SHOW VARIABLES LIKE 'innodb_log_files_in_group';

-- 복구 관련 설정
SHOW VARIABLES LIKE 'innodb_force_recovery';
```

**✨ 유용한 팁**
- Redo Log 크기를 적절히 설정 (너무 작으면 성능 저하)
- 정기적인 백업 수행
- 복구 테스트 정기 수행

---

### ✅ 4.2.7 InnoDB 버퍼 풀 (Buffer Pool)

#### 🎯 가장 중요한 캐시 영역

**버퍼 풀**은 MySQL 성능에 가장 중요한 요소입니다.

**역할:**
- 데이터 및 인덱스 페이지를 메모리에 보관
- 디스크 I/O 최소화
- 쿼리 성능 향상

**메모리 구조:**
```
InnoDB Buffer Pool
├── 데이터 페이지
├── 인덱스 페이지
├── Undo 페이지
├── Insert Buffer
└── Adaptive Hash Index
```

---

#### 📏 4.2.7.1 버퍼 풀 크기 조정

**관리 단위:**
- 버퍼 풀은 **128MB 청크 단위**로 관리
- 동적 크기 조정 가능 (MySQL 5.7+)

**설정 방법:**
```sql
-- 현재 버퍼 풀 크기 확인
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- 버퍼 풀 크기 설정 (예: 8GB)
SET GLOBAL innodb_buffer_pool_size = 8 * 1024 * 1024 * 1024;

-- 버퍼 풀 인스턴스 수 설정
SET GLOBAL innodb_buffer_pool_instances = 8;
```

**인스턴스 분할:**
- 기본값: 8개
- 1GB 미만이면 자동으로 1개
- **5GB 단위로 분할 권장** (잠금 경합 감소)

**크기 설정 가이드:**
- 서버 물리 메모리의 70-80% 권장
- 운영체제 및 다른 프로세스를 위한 메모리 확보 필요
- 너무 크게 설정하면 OOM(Out Of Memory) 발생 가능

**🌱 예시**
```sql
-- 32GB RAM 서버 기준
-- 버퍼 풀: 24GB (75%)
-- 인스턴스: 8개 (각 3GB)

SET GLOBAL innodb_buffer_pool_size = 24 * 1024 * 1024 * 1024;
SET GLOBAL innodb_buffer_pool_instances = 8;
```

---

#### 🏗️ 4.2.7.2 버퍼 풀의 구조

**주요 리스트:**
1. **LRU 리스트 (Least Recently Used)**
   - 최근에 사용된 페이지 관리
   - 자주 사용되는 페이지를 상단에 유지
   - 오래된 페이지는 하단으로 이동

2. **Free 리스트**
   - 사용 가능한 빈 페이지 관리
   - 새 페이지 로딩 시 Free 리스트에서 할당

3. **Flush 리스트**
   - 디스크에 쓰여야 할 Dirty Page 관리
   - 주기적으로 디스크에 플러시

**Dirty Page:**
- 메모리에서 변경되었지만 아직 디스크에 쓰이지 않은 페이지
- Checkpoint 시점에 디스크에 기록

**모니터링:**
```sql
-- 버퍼 풀 상태 확인
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- 페이지 상태 확인
SELECT 
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') AS total_pages,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_free') AS free_pages,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data') AS data_pages;
```

---

#### ⚙️ 4.2.7.3 버퍼 풀 운영 메커니즘

**읽기 시:**
1. 버퍼 풀에서 페이지 검색
2. 없으면 디스크에서 읽어서 버퍼 풀에 로딩
3. LRU 리스트에 추가

**쓰기 시:**
1. 버퍼 풀의 페이지 수정
2. Dirty Page로 표시
3. Flush 리스트에 등록
4. 주기적으로 디스크에 플러시

**플러시 전략:**
- **Adaptive Flush**: 워크로드에 따라 플러시 속도 조절
- **Checkpoint**: 주기적으로 Dirty Page를 디스크에 기록

**성능 최적화:**
- 버퍼 풀 히트율 모니터링 (99% 이상 권장)
- 적절한 버퍼 풀 크기 설정
- 디스크 I/O 최소화

**히트율 확인:**
```sql
-- 버퍼 풀 히트율 계산
SELECT 
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 
    AS buffer_pool_hit_ratio
FROM 
    (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_reads 
     FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') AS reads,
    (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_read_requests 
     FROM information_schema.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') AS requests;
```

---

#### 💾 4.2.7.4 버퍼 풀 백업/복구

**기능:**
- MySQL 종료 시 버퍼 풀의 페이지 정보를 파일로 저장
- 재시작 시 저장된 정보를 기반으로 버퍼 풀을 미리 로딩
- 워밍업 시간 단축

**설정:**
```sql
-- 종료 시 버퍼 풀 덤프
SET GLOBAL innodb_buffer_pool_dump_at_shutdown = ON;

-- 시작 시 버퍼 풀 로드
SET GLOBAL innodb_buffer_pool_load_at_startup = ON;

-- 덤프 파일 경로 확인
SHOW VARIABLES LIKE 'innodb_buffer_pool_dump_path';
```

**수동 덤프/로드:**
```sql
-- 수동 덤프
SET GLOBAL innodb_buffer_pool_dump_now = ON;

-- 수동 로드
SET GLOBAL innodb_buffer_pool_load_now = ON;

-- 로드 진행 상황 확인
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';
```

**✨ 활용법**
- 프로덕션 서버 재시작 시 워밍업 시간 단축
- 예상치 못한 재시작 후 빠른 성능 복구
- 스케줄링된 유지보수 시 활용

---

#### ⚠️ 4.2.7.5 버퍼 풀 로드 시 주의사항

**주의사항:**
1. **디스크 I/O 부하**
   - 버퍼 풀 로딩은 많은 디스크 읽기 발생
   - 대용량 버퍼 풀의 경우 시간이 오래 걸림
   - 시스템 부하 증가 가능

2. **로딩 시간**
   - 버퍼 풀 크기에 비례하여 로딩 시간 증가
   - 큰 버퍼 풀(수십 GB)의 경우 수 분 이상 소요 가능

3. **중단 방법**
   - 필요 시 로딩 중단 가능
   - 중단해도 서버는 정상 동작 (워밍업만 지연)

**중단 방법:**
```sql
-- 버퍼 풀 로딩 중단
SET GLOBAL innodb_buffer_pool_load_abort = ON;

-- 상태 확인
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';
```

**모니터링:**
```sql
-- 로딩 진행률 확인
SHOW STATUS LIKE 'Innodb_buffer_pool_load_incomplete';
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';

-- 로딩된 페이지 수 확인
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_loaded';
```

**권장사항:**
- 프로덕션 환경에서 로딩 중단 기능 활용
- 피크 시간대 피해서 로딩 수행
- 모니터링을 통한 로딩 상태 확인

