# MySQL 스토리지 엔진 아키텍처 (Vol.2)

### 4.2.8 Double Write Buffer

#### 📌 목적

디스크 페이지의 **"부분 쓰기(Partial Page Write)"** 문제 방지

#### ⚠️ 문제 상황

- 시스템 장애 시 더티 페이지의 **일부만 기록**되면 복구 불가능
- OS 레벨의 16KB 페이지 크기와 InnoDB의 16KB 페이지 크기가 매칭되지 않을 수 있음
- 디스크 쓰기 중 시스템 크래시 시 페이지 손상 가능

#### ✅ 해결 방식

```
[더티 페이지] → [Double Write Buffer] → [데이터 파일]
                      ↓ (128개 페이지, 약 2MB)
                   [시스템 테이블스페이스]
```

1. **1단계**: 더티 페이지를 먼저 **Double Write Buffer**(128개 페이지, 약 2MB)에 기록
2. **2단계**: 이후 실제 **데이터 파일**에 최종 기록
3. **장애 복구**: Double Write Buffer와 리두 로그(Redo Log)를 비교해 손상된 페이지 복원 가능

#### 💡 특징

- **디스크 쓰기 부하**: 약간 증가 (약 5~10% 성능 저하)
- **데이터 안정성**: 대폭 향상
- **저장 위치**: 시스템 테이블스페이스(`ibdata1`)의 연속된 영역

#### 🔧 설정

- **비활성화**: `innodb_doublewrite = OFF` (권장하지 않음)
- **모니터링**: `SHOW STATUS LIKE 'Innodb_dblwr%'`

#### 💬 실무 팁

- SSD 환경에서는 Double Write Buffer 성능 저하가 적음
- 매우 안정적인 환경이라도 데이터 무결성을 위해 비활성화는 권장하지 않음

---

### 4.2.9 Undo Log (언두 로그)

#### 📌 주요 기능

1. **트랜잭션 롤백**: 변경 전 상태로 복구
2. **MVCC 구현**: 일관된 읽기 제공 (읽기 일관성 보장)
3. **암시적 잠금 관리**: 트랜잭션의 일관성 유지

#### 📁 저장 위치

- **Undo Tablespace**: `undo_001.ibu`, `undo_002.ibu` 등
- MySQL 8.0부터는 별도 파일로 분리 관리

#### 🔍 Undo Log의 종류

| 종류            | 용도               | 설명                               |
| --------------- | ------------------ | ---------------------------------- |
| **Insert Undo** | INSERT 롤백        | 새 레코드 삽입 시 롤백용 백업      |
| **Update Undo** | UPDATE/DELETE 롤백 | 기존 레코드 수정/삭제 전 상태 백업 |

#### 🔄 Purge Thread

- 커밋 후, 더 이상 참조되지 않는 Undo 레코드를 삭제
- 백그라운드에서 자동 실행
- `innodb_purge_threads`로 스레드 수 설정 (기본값: 4)

#### ⚠️ Undo 크기 증가 원인

1. **장시간 열린 트랜잭션**: 오래된 스냅샷을 유지해야 함
2. **Purge 지연**: 삭제 작업이 느려서 Undo 레코드가 누적
3. **롱 트랜잭션**: 대량 데이터 변경 후 커밋하지 않음

#### 🔧 모니터링

```sql
-- Undo 테이블스페이스 크기 확인
SELECT
    tablespace_name,
    file_name,
    total_extents * extent_size / 1024 / 1024 AS size_mb
FROM information_schema.FILES
WHERE tablespace_name LIKE 'innodb_undo%';

-- Purge 지연 확인
SHOW VARIABLES LIKE 'innodb_purge_threads';
SHOW ENGINE INNODB STATUS\G  -- History list length 확인
```

#### 💬 실무 팁

- Undo 테이블스페이스가 급격히 증가하면 장시간 트랜잭션 확인 필요
- `History list length`가 100만을 넘으면 주의 필요

---

### 4.2.10 Change Buffer (체인지 버퍼)

#### 📌 목적

인덱스 페이지를 **즉시 디스크에 반영하지 않고** 메모리에 임시 보관하여 **디스크 I/O 감소**

#### ✅ 적용 대상

- **비유니크 보조 인덱스**만 대상
- **유니크 인덱스**: 중복 체크가 필요하므로 즉시 확인 필요 → 사용 불가
- **프라이머리 키**: 클러스터링 인덱스이므로 변경 버퍼 미사용

#### 🔄 동작 방식

```
1. INSERT/DELETE/UPDATE 발생
   ↓
2. 인덱스 변경 내용을 Change Buffer에 임시 저장
   ↓
3. 백그라운드 Merge Thread가 실제 페이지와 병합(Flush)
   ↓
4. 디스크 I/O 감소 → 쓰기 성능 향상
```

#### 💡 효과

- **디스크 I/O 감소**: 랜덤 I/O → 순차 I/O로 변환 가능
- **쓰기 성능 향상**: 특히 보조 인덱스가 많은 테이블에서 효과적
- **버퍼 풀 효율**: 인덱스 페이지를 버퍼 풀에서 제거할 수 있음

#### 🔧 설정 변수

```sql
-- Change Buffer 버퍼링 유형 설정
SET GLOBAL innodb_change_buffering = all | inserts | deletes | changes | purges | none;

-- all: INSERT, DELETE, UPDATE 모두 버퍼링 (기본값)
-- inserts: INSERT만 버퍼링
-- deletes: DELETE만 버퍼링
-- changes: INSERT, DELETE 버퍼링
-- purges: DELETE로 인한 인덱스 제거 작업 버퍼링
-- none: 비활성화
```

#### 📊 크기 제한

- Change Buffer는 **버퍼 풀 메모리의 최대 25%**까지 사용 가능
- `innodb_change_buffer_max_size = 25` (기본값)

#### 🔍 모니터링

```sql
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_%';
-- Innodb_buffer_pool_pages_insert
-- Innodb_buffer_pool_pages_delete_mark
-- Innodb_buffer_pool_pages_delete
```

#### 💬 실무 팁

- **읽기 중심 애플리케이션**: `all` 또는 `inserts` 권장
- **쓰기 중심 애플리케이션**: `none`으로 설정하여 즉시 반영
- **보조 인덱스가 많은 테이블**: Change Buffer 효과 극대화

---

### 4.2.11 Redo Log & Log Buffer

#### 📌 Redo Log (리두 로그)

##### 개념

트랜잭션의 변경 내용을 **"재적용"**할 수 있게 기록하는 **물리적 로그**

##### 주요 기능

- 장애 복구 시, **커밋된 변경사항**을 다시 반영
- ACID 특성 중 **Durability(지속성)** 보장
- 데이터 페이지의 변경 내용을 바이트 단위로 기록

##### 구조

- **순환 구조(Circular Buffer)**: 로그 파일이 가득 차면 처음부터 다시 사용
- **파일**: `ib_logfile0`, `ib_logfile1`, ... (기본 2개)
- **크기/개수 제어**:
  - `innodb_log_file_size`: 각 로그 파일 크기 (기본: 48MB)
  - `innodb_log_files_in_group`: 로그 파일 개수 (기본: 2)

##### LSN (Log Sequence Number)

- Redo Log의 각 레코드에 부여되는 고유 번호
- 데이터베이스 복구 시 어디까지 적용했는지 추적

#### 📌 Log Buffer (로그 버퍼)

##### 개념

Redo Log를 **임시로 저장하는 메모리 버퍼**

##### Flush 시점

```sql
SET GLOBAL innodb_flush_log_at_trx_commit = 0 | 1 | 2;
```

| 값    | 동작                         | 성능    | 안정성  | 설명                                           |
| ----- | ---------------------------- | ------- | ------- | ---------------------------------------------- |
| **0** | 1초마다 Flush                | ⚡ 빠름 | ⚠️ 낮음 | 1초마다 디스크에 기록, 크래시 시 최대 1초 분실 |
| **1** | 매 트랜잭션마다 Flush        | 🐌 느림 | ✅ 높음 | **기본값**, 매 커밋마다 디스크에 기록          |
| **2** | 매 커밋마다 OS 버퍼에만 기록 | ⚡ 빠름 | ⚠️ 중간 | OS가 Flush, OS 크래시 시 데이터 손실 가능      |

##### 권장 설정

- **데이터 무결성 중요**: `1` (기본값)
- **성능 우선 (데이터 손실 허용 가능)**: `2`
- **데이터 손실 허용 불가**: 절대 `0` 사용 금지

#### 🔄 Checkpoint (체크포인트)

##### 개념

Redo Log의 특정 지점을 **디스크에 반영 완료**로 표시

##### 역할

1. **로그 재사용**: 반영된 로그 영역을 재사용 가능하게 함
2. **복구 시간 단축**: 체크포인트 이후부터만 복구하면 됨
3. **디스크 동기화**: 버퍼 풀의 더티 페이지를 주기적으로 디스크에 기록

##### Checkpoint 종류

| 종류                 | 설명                                            |
| -------------------- | ----------------------------------------------- |
| **Sharp Checkpoint** | 모든 더티 페이지를 디스크에 기록 (서버 종료 시) |
| **Fuzzy Checkpoint** | 일부 더티 페이지만 점진적으로 기록 (일반 동작)  |

#### 🔍 모니터링

```sql
-- Redo Log 크기 확인
SHOW VARIABLES LIKE 'innodb_log%';

-- Log Buffer 상태
SHOW VARIABLES LIKE 'innodb_log_buffer_size';

-- Redo Log 사용량 (모니터링 도구 필요)
SHOW ENGINE INNODB STATUS\G
-- Log sequence number, Log flushed up to 확인
```

#### 💬 실무 팁

- **Redo Log 크기**: 트랜잭션 처리량에 따라 조정 필요
  - 너무 작으면: 빈번한 Checkpoint로 성능 저하
  - 너무 크면: 복구 시간 증가
  - 일반적으로 **256MB ~ 2GB** 권장
- **로그 파일 크기 변경**: 서버 재시작 필요 (온라인 변경 불가)
- **트랜잭션 로그 모니터링**: 정기적으로 로그 사용률 확인

---

### 4.2.12 Adaptive Hash Index (AHI)

#### 📌 개념

자주 읽히는 인덱스 키 패턴을 감지해, **메모리 상에 해시 인덱스를 자동 생성**

#### 🔄 동작 방식

```
1. B-Tree 인덱스 검색 중 특정 페이지 접근이 반복됨
   ↓
2. 자동으로 해시 인덱스 생성
   ↓
3. 이후 B-Tree 탐색 없이 해시로 바로 찾기 가능
   ↓
4. 성능 향상 ⚡
```

#### ✅ 장점

- **빠른 조회**: B-Tree 탐색 비용 없이 O(1) 시간에 접근
- **자동 최적화**: MySQL이 자동으로 생성/관리
- **메모리 효율**: 버퍼 풀의 일부만 사용

#### ⚠️ 단점 및 주의사항

- **테이블 변경 시 자동 무효화**: INSERT/UPDATE/DELETE 시 해시 인덱스 재생성 필요
- **Lock 경합 가능**: 쓰기 중심 OLTP 환경에서는 오히려 Lock 경합 발생 가능
- **메모리 사용**: 버퍼 풀 메모리의 일부 사용

#### 🔧 제어 방법

```sql
-- 현재 상태 확인
SHOW VARIABLES LIKE 'innodb_adaptive_hash_index';

-- 비활성화
SET GLOBAL innodb_adaptive_hash_index = OFF;

-- 활성화 (기본값: ON)
SET GLOBAL innodb_adaptive_hash_index = ON;
```

#### 💬 실무 팁

- **읽기 중심 환경**: AHI 활성화 권장 (기본값 유지)
- **쓰기 중심 OLTP**: Lock 경합이 발생하면 비활성화 고려
- **모니터링**: `SHOW ENGINE INNODB STATUS`에서 `Hash table size` 확인

---

### 4.2.13 InnoDB vs MyISAM / MEMORY 엔진 비교

#### 📊 상세 비교표

| 항목              | InnoDB                   | MyISAM                  | MEMORY                     |
| ----------------- | ------------------------ | ----------------------- | -------------------------- |
| **트랜잭션**      | ✅ 지원                  | ❌ 미지원               | ❌ 미지원                  |
| **외래키**        | ✅ 지원                  | ❌ 미지원               | ❌ 미지원                  |
| **잠금 단위**     | Row-level                | Table-level             | Table-level                |
| **복구 기능**     | Redo/Undo 기반 완전 복구 | 제한적                  | 데이터 손실                |
| **캐시 구조**     | 버퍼 풀 (데이터+인덱스)  | Key Cache (인덱스 전용) | 메모리 기반                |
| **쓰기 안정성**   | Double Write Buffer      | OS 의존                 | 서버 재시작 시 데이터 손실 |
| **동시성**        | 높음 (MVCC)              | 낮음                    | 낮음                       |
| **데이터 저장**   | 테이블스페이스           | 별도 파일 (.MYD, .MYI)  | 메모리 (휘발성)            |
| **커밋 시점**     | 즉시 반영                | 명시적 플러시 필요      | -                          |
| **사용 시나리오** | OLTP, 일반 목적          | 읽기 중심, 통계         | 임시 테이블, 빠른 조회     |

#### 💡 선택 기준

##### InnoDB 사용 권장 상황

- 트랜잭션이 필요한 애플리케이션
- 외래키 제약조건 필요
- 높은 동시성 처리 필요
- 데이터 무결성이 중요한 경우
- **MySQL 5.5.5 이후 기본 스토리지 엔진**

##### MyISAM 사용 고려 상황 (현재 거의 사용 안 함)

- 읽기 중심의 단순한 애플리케이션
- 트랜잭션이 필요 없는 경우
- MySQL 5.7 이하 버전의 특수한 경우

##### MEMORY 사용 고려 상황

- 임시 테이블
- 매우 빠른 조회가 필요한 경우
- 데이터 손실 허용 가능한 경우

---

## 4.3 MyISAM 스토리지 엔진 아키텍처

### 4.3.1 키 캐시 (Key Cache)

#### 📌 개념

MyISAM의 **인덱스 전용 캐시**

#### 🔍 특징

- **데이터 블록 캐싱**: OS 파일 시스템 캐시가 담당
- **분리 관리**:
  - 데이터 페이지 → OS 캐시
  - 인덱스 → Key Cache
- **버퍼 풀과의 차이**: InnoDB는 데이터+인덱스를 함께 관리

#### 🔧 설정

```sql
-- 키 캐시 크기 설정
SET GLOBAL key_buffer_size = 256 * 1024 * 1024;  -- 256MB

-- 키 캐시 사용률 확인
SHOW STATUS LIKE 'Key%';
```

---

### 4.3.2 운영체제 캐시 및 버퍼

#### 📌 아키텍처 차이

| 엔진       | 캐시 관리 방식           | 장점                        | 단점                  |
| ---------- | ------------------------ | --------------------------- | --------------------- |
| **InnoDB** | 자체 캐시 관리 (버퍼 풀) | 예측 가능한 성능, 제어 가능 | 메모리 중복 사용 가능 |
| **MyISAM** | OS에 위임                | 메모리 효율적, 구현 단순    | OS 설정에 의존적      |

#### 💡 특징

- MyISAM은 자체 버퍼를 거의 사용하지 않음
- OS 레벨의 파일 시스템 캐시에 의존
- 메모리 효율적이지만 성능 예측이 어려움

---

### 4.3.3 데이터 파일 & 프라이머리 키 구조

#### 📁 파일 구조

- **인덱스 파일**: `.MYI` (MyISAM Index)
- **데이터 파일**: `.MYD` (MyISAM Data)
- **테이블 정의**: `.frm` (MySQL 8.0 이전)

#### 🔍 인덱스 구조 비교

| 특징              | InnoDB                                          | MyISAM                                     |
| ----------------- | ----------------------------------------------- | ------------------------------------------ |
| **프라이머리 키** | 클러스터링 인덱스 (데이터가 인덱스와 함께 저장) | 비클러스터형 인덱스 (인덱스와 데이터 분리) |
| **참조 방식**     | 프라이머리 키로 직접 데이터 접근                | 인덱스 → 데이터 파일 포인터 → 실제 데이터  |

#### ⚠️ 복구

- **테이블 손상 시**: `myisamchk` 툴 필요
- **복구 명령어**:
  ```bash
  myisamchk --recover --quick /path/to/table.MYI
  ```
- **자동 복구**: `myisam_recover_options` 설정 가능

---

## 4.4 MySQL 로그 파일

### 📋 로그 종류 및 목적

| 로그 종류            | 목적                                  | 파일 이름                        | 주요 설정                           |
| -------------------- | ------------------------------------- | -------------------------------- | ----------------------------------- |
| **에러 로그**        | 서버 시작/종료, 오류, 경고 기록       | `mysqld.err` 또는 `hostname.err` | `log_error`                         |
| **일반 쿼리 로그**   | 실행된 모든 SQL 기록                  | `hostname.log`                   | `general_log`, `general_log_file`   |
| **슬로우 쿼리 로그** | 실행 시간이 긴 쿼리 분석용            | `hostname-slow.log`              | `slow_query_log`, `long_query_time` |
| **바이너리 로그**    | 복제 및 PITR (Point-In-Time Recovery) | `mysql-bin.000001`, ...          | `log_bin`, `binlog_format`          |
| **릴레이 로그**      | 슬레이브 서버의 복제용                | `relay-bin.000001`, ...          | `relay_log`                         |
| **트랜잭션 로그**    | InnoDB 리두 로그                      | `ib_logfile0`, `ib_logfile1`     | `innodb_log_file_size`              |

### 🔍 주요 로그 상세

#### 4.4.1 에러 로그 (Error Log)

##### 설정

```sql
-- 에러 로그 파일 위치 확인
SHOW VARIABLES LIKE 'log_error';

-- 콘솔 출력 포함
SET GLOBAL log_error_verbosity = 1 | 2 | 3;  -- 1: 에러만, 2: 에러+경고, 3: 모두
```

##### 확인 방법

```bash
# 에러 로그 실시간 모니터링
tail -f /var/log/mysql/error.log

# 최근 에러 확인
grep -i error /var/log/mysql/error.log | tail -20
```

##### 주요 내용

- 서버 시작/종료 이벤트
- 크리티컬 에러
- 경고 메시지
- 정상 동작 메시지

---

#### 4.4.2 일반 쿼리 로그 (General Query Log)

##### 설정

```sql
-- 활성화
SET GLOBAL general_log = ON;
SET GLOBAL general_log_file = '/var/log/mysql/general.log';

-- 비활성화
SET GLOBAL general_log = OFF;
```

##### 특징

- **모든 SQL 쿼리** 기록
- 성능 오버헤드 큼 (프로덕션 환경에서는 주의)
- 디버깅 및 트러블슈팅용

---

#### 4.4.3 슬로우 쿼리 로그 (Slow Query Log)

##### 설정

```sql
-- 활성화
SET GLOBAL slow_query_log = ON;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';

-- 슬로우 쿼리 기준 시간 설정 (초 단위)
SET GLOBAL long_query_time = 2;  -- 2초 이상 실행된 쿼리 기록

-- 인덱스를 사용하지 않은 쿼리도 기록
SET GLOBAL log_queries_not_using_indexes = ON;

-- 로그 슬로우 쿼리 기록
SET GLOBAL log_slow_admin_statements = ON;
```

##### 분석 도구

```bash
# mysqldumpslow 사용
mysqldumpslow /var/log/mysql/slow.log

# pt-query-digest 사용 (Percona Toolkit)
pt-query-digest /var/log/mysql/slow.log
```

##### 활용

- 성능 튜닝의 출발점
- 인덱스 최적화 포인트 파악
- 쿼리 패턴 분석

---

#### 4.4.4 바이너리 로그 (Binary Log)

##### 설정

```sql
-- 바이너리 로그 활성화
SET GLOBAL log_bin = ON;

-- 바이너리 로그 포맷 설정
SET GLOBAL binlog_format = STATEMENT | ROW | MIXED;
-- STATEMENT: SQL 문 기록 (복제 시 데이터 불일치 가능)
-- ROW: 행 단위 변경 기록 (기본값, 안전함)
-- MIXED: 두 가지 혼합
```

##### 용도

- **복제 (Replication)**: 마스터-슬레이브 복제
- **PITR (Point-In-Time Recovery)**: 특정 시점으로 복구
- **데이터 변경 이력**: 모든 변경사항 기록

##### 관리

```sql
-- 바이너리 로그 목록 확인
SHOW BINARY LOGS;

-- 현재 바이너리 로그 확인
SHOW MASTER STATUS;

-- 바이너리 로그 삭제 (주의!)
PURGE BINARY LOGS TO 'mysql-bin.000010';  -- 해당 로그 이전 삭제
PURGE BINARY LOGS BEFORE DATE('2024-01-01');  -- 날짜 기준 삭제
```