# Database / MySQL 핵심 정리

---

## 1. 인덱스 (Index)

**B-Tree 인덱스** (MySQL InnoDB 기본)
- 데이터를 정렬된 트리 구조로 저장
- `=`, `<`, `>`, `BETWEEN`, `LIKE 'abc%'` 등에 효과적
- 범위 검색(Range Scan)에 적합

**Hash 인덱스**
- 동등 비교(`=`)에만 적합, 범위 검색 불가
- MySQL Memory 엔진에서 사용

**인덱스가 사용되지 않는 경우**
```sql
WHERE YEAR(created_at) = 2024   -- 컬럼에 함수 적용
WHERE name LIKE '%abc'          -- 앞에 와일드카드
WHERE col1 OR col2              -- OR 조건 (일반적으로)
-- 카디널리티가 낮은 컬럼 (예: gender)
-- 데이터가 전체의 20~30% 이상 조회되는 경우
```

**복합 인덱스**: 가장 왼쪽 컬럼부터 순서대로 사용됨 (Leftmost Prefix Rule)
```sql
INDEX (a, b, c)
-- a 단독, a+b, a+b+c 는 인덱스 사용 가능
-- b 단독, c 단독, b+c 는 인덱스 사용 불가
```

**클러스터드 인덱스 vs 논클러스터드 인덱스**
- **클러스터드**: 실제 데이터가 인덱스 순서대로 저장. InnoDB의 Primary Key
- **논클러스터드**: 인덱스는 별도 저장, 실제 데이터 위치(PK)를 가리킴. Secondary Index

---

## 2. 트랜잭션과 ACID

| 속성 | 설명 |
|------|------|
| **Atomicity (원자성)** | 트랜잭션 내 작업은 전부 성공 또는 전부 실패 |
| **Consistency (일관성)** | 트랜잭션 전후로 DB 제약 조건이 유지됨 |
| **Isolation (격리성)** | 동시 실행 트랜잭션들이 서로 간섭하지 않음 |
| **Durability (지속성)** | 커밋된 데이터는 장애 후에도 보존됨 |

---

## 3. 트랜잭션 격리 수준

| 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|------|-----------|---------------------|--------------|
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 방지 | 발생 | 발생 |
| REPEATABLE READ | 방지 | 방지 | 발생(InnoDB는 방지) |
| SERIALIZABLE | 방지 | 방지 | 방지 |

- **MySQL InnoDB 기본**: `REPEATABLE READ`
- **Dirty Read**: 커밋 안 된 데이터 읽기
- **Non-Repeatable Read**: 같은 쿼리 두 번 실행 시 결과 다름
- **Phantom Read**: 범위 쿼리 시 없던 행이 생기거나 사라짐

---

## 4. Lock 종류

**낙관적 락 (Optimistic Lock)**
- 충돌이 드물다고 가정, 실제 충돌 시 재시도
- `version` 컬럼으로 구현: UPDATE WHERE version = ?
- 게임에서 아이템 수량, 재화 등에 활용

**비관적 락 (Pessimistic Lock)**
- 충돌 가능성이 높다고 가정, 미리 락 획득
- `SELECT ... FOR UPDATE`
- 데드락 주의

**InnoDB 락 유형**
- **Shared Lock (S)**: 읽기 락. 다른 S락과 공존 가능
- **Exclusive Lock (X)**: 쓰기 락. 다른 락과 공존 불가
- **Gap Lock**: 인덱스 범위 사이의 "틈"에 거는 락. Phantom Read 방지
- **Next-Key Lock**: Row Lock + Gap Lock 조합

---

## 5. 실행 계획 (EXPLAIN)

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@test.com';
```

| 컬럼 | 설명 |
|------|------|
| type | 접근 방식. `const` > `eq_ref` > `ref` > `range` > `index` > `ALL` |
| key | 실제 사용된 인덱스 |
| rows | 예상 스캔 행 수 |
| Extra | `Using index`(커버링), `Using filesort`(주의), `Using temporary`(주의) |

- **type=ALL**: 풀 테이블 스캔 → 인덱스 추가 검토
- **Using filesort**: ORDER BY를 위해 추가 정렬 발생
- **Using index**: 인덱스만으로 쿼리 처리 (커버링 인덱스)

---

## 6. InnoDB 스토리지 엔진

**InnoDB vs MyISAM**

| 항목 | InnoDB | MyISAM |
|------|--------|--------|
| 트랜잭션 | 지원 | 미지원 |
| 외래키 | 지원 | 미지원 |
| 락 단위 | Row-level | Table-level |
| 장애 복구 | 자동 (redo log) | 수동 |
| 전문 검색 | 지원 | 지원 |

**InnoDB 내부 구조**
- **Buffer Pool**: 자주 사용하는 데이터/인덱스 페이지 캐시. 전체 메모리의 70~80% 권장
- **Redo Log**: 변경 사항 기록, 크래시 복구에 사용
- **Undo Log**: 이전 버전 데이터 보관, MVCC와 롤백에 사용
- **MVCC**: 잠금 없이 일관된 읽기 제공. 변경 전 버전을 Undo Log에서 읽음

---

## 7. 복제 (Replication)

**MySQL 복제 구조**
```
Primary (Master) → Binary Log → Relay Log → Replica (Slave)
```

- **비동기 복제**: 기본값. Primary는 Replica의 적용 완료를 기다리지 않음
- **반동기 복제**: 최소 1개 Replica가 수신 확인 후 커밋
- **GTID**: 글로벌 트랜잭션 ID. 복제 위치 추적 및 페일오버 간소화

**용도**
- 읽기 부하 분산 (Primary: 쓰기, Replica: 읽기)
- 백업, 장애 복구

---

## 8. 쿼리 최적화 팁

```sql
-- 1. SELECT * 피하기 → 필요한 컬럼만
SELECT id, name FROM users WHERE ...

-- 2. WHERE 절에 인덱스 컬럼을 가공하지 않기
-- 나쁨:
WHERE DATE(created_at) = '2024-01-01'
-- 좋음:
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'

-- 3. 페이지네이션 개선 (OFFSET 큰 경우)
-- 나쁨:
SELECT * FROM logs ORDER BY id LIMIT 100 OFFSET 1000000
-- 좋음:
SELECT * FROM logs WHERE id > 1000000 ORDER BY id LIMIT 100

-- 4. 커버링 인덱스 활용
-- users(name, email) 인덱스가 있을 때
SELECT name, email FROM users WHERE name = 'kim'  -- 테이블 접근 불필요

-- 5. JOIN 시 인덱스 확인
-- ON 절의 양쪽 컬럼에 인덱스 있어야 함
```

---

## 9. 게임 서버에서 자주 쓰는 패턴

**유저 재화/수치 갱신 (동시성 처리)**
```sql
-- 원자적 업데이트
UPDATE users SET gold = gold - 100 WHERE user_id = ? AND gold >= 100

-- 낙관적 락
UPDATE users SET gold = gold - 100, version = version + 1 
WHERE user_id = ? AND version = ? AND gold >= 100
```

**랭킹 조회**
```sql
-- 인덱스 설계: (score DESC, user_id)
SELECT user_id, score, RANK() OVER (ORDER BY score DESC) as rank
FROM user_scores
LIMIT 100;
```

**파티셔닝 (대용량 로그 테이블)**
```sql
-- Range 파티셔닝으로 날짜별 분리
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
)
```

---

## 10. 자주 나오는 면접 질문

**Q. N+1 문제란?**
1번의 쿼리로 N개의 결과를 가져온 후, 각 결과에 대해 추가 쿼리 N번이 발생하는 문제. JOIN이나 IN절로 해결.

**Q. 인덱스가 많으면 좋은가?**
아니다. 인덱스는 쓰기(INSERT/UPDATE/DELETE) 시 추가 비용이 발생하고 저장 공간도 사용. 자주 조회되는 컬럼에만 설정.

**Q. 샤딩(Sharding)이란?**
데이터를 여러 DB 서버에 수평 분산. User ID 기반 샤딩 등. 조인이 어렵고 재샤딩 비용이 높음.

**Q. 슬로우 쿼리 찾는 방법?**
`slow_query_log = ON`, `long_query_time = 1` 설정 후 슬로우 쿼리 로그 분석. `mysqldumpslow`나 `pt-query-digest` 도구 사용.
