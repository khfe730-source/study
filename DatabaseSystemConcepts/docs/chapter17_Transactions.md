# Chapter 17: Transactions

## 개요

트랜잭션은 데이터베이스의 논리적 작업 단위다. ACID 속성(Atomicity·Consistency·Isolation·Durability), 트랜잭션 상태 모델, 스케줄, 직렬가능성(Conflict Serializability), 회복 가능 스케줄, 격리 수준을 다룬다.

---

## 17.1 트랜잭션 개념과 ACID

**트랜잭션**: `begin transaction` ~ `end transaction` 사이의 모든 연산으로 구성된 단원.

### ACID 속성

| 속성 | 설명 | 담당 시스템 구성요소 |
|------|------|---------------------|
| **Atomicity (원자성)** | 모두 반영되거나 전혀 반영되지 않음 | 복구 시스템 (Recovery System) |
| **Consistency (일관성)** | 트랜잭션이 DB를 일관된 상태에서 일관된 상태로 전이 | 응용 프로그래머 |
| **Isolation (고립성)** | 동시 실행 시 다른 트랜잭션의 중간 상태를 볼 수 없음 | 동시성 제어 시스템 |
| **Durability (영속성)** | 커밋된 트랜잭션의 변경은 장애 후에도 유지 | 복구 시스템 |

```
자금 이체 트랜잭션 Ti (A → B, $50):
  read(A);  A := A − 50;  write(A)     ← 여기서 충돌 → A = 950, B = 2000 (불일치!)
  read(B);  B := B + 50;  write(B)
                                         원자성: 둘 다 반영 or 둘 다 미반영
```

---

## 17.2 단순 트랜잭션 모델

기본 연산:
- `read(X)`: 디스크 → 메모리 버퍼로 X 읽기
- `write(X)`: 메모리 버퍼의 X → 디스크에 기록 (즉시 쓰기 가정)

---

## 17.3 저장 구조

| 종류 | 특성 | 예시 |
|------|------|------|
| **휘발성 저장** | 시스템 충돌 시 데이터 손실 | 주 메모리, 캐시 |
| **비휘발성 저장** | 충돌 후에도 생존, 디스크 오류에는 취약 | 자기 디스크, 플래시 |
| **안정 저장** | 절대 손실 없음 (이론적) | 미러링 디스크, RAID |

**안정 저장 구현**: 독립적 오류 모드를 가진 여러 비휘발성 저장 매체에 복제.

---

## 17.4 트랜잭션 원자성과 영속성

### 트랜잭션 상태 다이어그램

```
      ┌──────────────────────────────────────────────────────┐
      │ active → partially committed → committed (성공)      │
      │    └─────────► failed ─────────► aborted (실패)      │
      └──────────────────────────────────────────────────────┘

Active:              초기 상태. 실행 중.
Partially committed: 마지막 문장 실행 완료. 아직 디스크에 쓰지 않음.
Committed:           변경 사항이 안정 저장에 기록됨. 복구 가능.
Failed:              정상 실행 불가 (하드웨어 오류, 논리 오류 등).
Aborted:             롤백 완료. 재시작 or 종료 가능.
```

**로그(Log)**: 모든 DB 변경을 디스크에 기록하는 시퀀스. 충돌 후 redo/undo를 위해 반드시 필요.

---

## 17.5 트랜잭션 고립성

### 동시 실행의 이유

1. **처리량 향상**: CPU와 I/O 병렬 활용
2. **대기 시간 감소**: 짧은 트랜잭션이 긴 트랜잭션을 기다리지 않아도 됨

### 스케줄 (Schedule)

여러 트랜잭션의 연산이 실행되는 시간 순서.

**직렬 스케줄(Serial Schedule)**: 한 트랜잭션이 완전히 완료된 후 다음 트랜잭션 시작.

```
Schedule 1 (직렬: T1 → T2):
T1: read(A), A:=A-50, write(A), read(B), B:=B+50, write(B), commit
T2: read(A), temp:=A*0.1, A:=A-temp, write(A), read(B), B:=B+temp, write(B), commit
→ A+B 합계 유지됨 (일관됨)

Schedule 4 (비직렬, 불일치):
T1: read(A), A:=A-50
T2: read(A) ← T1의 변경값 읽기
T2: A:=A-temp, write(A)
T1: write(A) ← T2의 write 덮어씀
→ A+B 합계 불일치 발생
```

---

## 17.6 직렬가능성 (Serializability)

### 충돌(Conflict)

두 연산이 **다른 트랜잭션**에 속하고, **동일 데이터**를 접근하며, **적어도 하나가 write**이면 충돌.

| 연산 쌍 | 충돌 여부 |
|---------|---------|
| read(Q), read(Q) | ❌ (순서 무관) |
| read(Q), write(Q) | ✅ |
| write(Q), read(Q) | ✅ |
| write(Q), write(Q) | ✅ |

### 충돌 동치(Conflict Equivalent)

충돌하지 않는 연산의 순서를 교환하여 S를 S'로 변환 가능 → 두 스케줄은 충돌 동치.

### 충돌 직렬가능성(Conflict Serializability)

어떤 직렬 스케줄과 충돌 동치이면 **충돌 직렬가능**.

### 선행 그래프(Precedence Graph)로 판별

```
노드: 각 트랜잭션
엣지: Ti → Tj  if Ti가 Q에 write 하고 Tj가 그 Q를 read 하거나,
              Ti가 Q를 read 하고 Tj가 그 Q를 write 하거나,
              Ti, Tj 모두 Q에 write 할 때 (Ti가 먼저)

사이클 없음 → 충돌 직렬가능
사이클 있음 → 충돌 직렬불가능
```

```
예: Schedule 4
  T1 → T2  (T1이 A를 read하기 전에 T2가 A를 write)
  T2 → T1  (T2가 B를 read하기 전에 T1이 B를 write)
  → 사이클 존재 → 충돌 직렬가능 아님
```

**위상 정렬(Topological Sort)**: 사이클 없는 선행 그래프에서 직렬화 순서를 추출.

### 뷰 직렬가능성(View Serializability)

충돌 동치보다 약한 동치. **블라인드 라이트(Blind Write)**를 포함하는 스케줄을 허용. NP-완전 판별이라 실용적이지 않음.

---

## 17.7 트랜잭션 고립성과 원자성

### 회복 가능 스케줄(Recoverable Schedule)

Tj가 Ti의 write 결과를 read했다면, Tj의 커밋은 Ti의 커밋 이후에 발생해야 함.

```
비회복 가능 스케줄 (Schedule 9):
T6: write(A) → T7: read(A) → T7: commit → T6: fail!!
  T7는 이미 커밋됐지만 T6 중단으로 T7의 read는 더티 읽기
  → T7를 undo 불가 → 불일치 허용 불가
```

### 연쇄 없는 스케줄(Cascadeless Schedule)

Tj가 Ti의 write 결과를 read하기 **전에** Ti가 커밋해야 함.

```
연쇄 롤백 (Cascading Rollback):
T8: write(A)
T9: read(A), write(A)  ← T8에 의존
T10: read(A)           ← T9에 의존
T8 abort → T9 abort → T10 abort  (연쇄 롤백)
```

**모든 연쇄 없는 스케줄은 회복 가능**하지만, 역은 성립하지 않음.

---

## 17.8 SQL 격리 수준

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom |
|---------|-----------|---------------------|---------|
| **Read Uncommitted** | 허용 | 허용 | 허용 |
| **Read Committed** | ❌ | 허용 | 허용 |
| **Repeatable Read** | ❌ | ❌ | 허용 |
| **Serializable** | ❌ | ❌ | ❌ |

```sql
-- 격리 수준 설정
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- 대부분 DB의 기본값

-- 트랜잭션 시작 (자동 커밋 해제)
START TRANSACTION;  -- MySQL, PostgreSQL
BEGIN TRANSACTION;  -- SQL Server
-- Oracle/PostgreSQL: BEGIN

COMMIT;
ROLLBACK;
```

---

## 17.9 격리 수준 구현 개요

### 17.9.1 잠금(Locking)

**2단계 잠금(Two-Phase Locking)**:
1. 성장 단계(Growing): 잠금 획득만 가능
2. 수축 단계(Shrinking): 잠금 해제만 가능

```
Shared Lock (S): 읽기 전용
Exclusive Lock (X): 읽기/쓰기

잠금 호환성:
     S    X
  S  ✅   ❌
  X  ❌   ❌
```

### 17.9.2 타임스탬프 (Timestamps)

각 트랜잭션에 고유 타임스탬프 부여. 충돌 시 타임스탬프 순서를 위반하면 롤백.

### 17.9.3 다중 버전과 스냅샷 격리 (Snapshot Isolation)

트랜잭션 시작 시점의 DB 스냅샷을 제공. 읽기는 절대 대기하지 않음.

```
장점: 읽기 트랜잭션이 쓰기 트랜잭션 블로킹 없음
단점: Write Skew 이상현상 가능 → 완전한 직렬가능성 미보장
```

---

## 17.10 SQL 트랜잭션과 팬텀 현상

**팬텀 현상(Phantom Phenomenon)**: 같은 튜플을 공유하지 않더라도 INSERT/DELETE 때문에 발생하는 논리적 충돌.

```sql
-- T30: 물리학과 교수 수 조회
SELECT count(*) FROM instructor WHERE dept_name = 'Physics';

-- T31: 새 물리학과 교수 삽입
INSERT INTO instructor VALUES (11111, 'Feynman', 'Physics', 94000);

-- T30, T31이 동시에 실행 → 충돌 없는 튜플이지만 논리적으로 충돌
-- → 인덱스 리프 노드 잠금으로 해결 (Index-Locking Protocol)
```

**술어 잠금(Predicate Locking)**: `salary > 90000` 같은 조건에 잠금 획득. 팬텀 방지하지만 구현 복잡.

---

## 17.11 핵심 요약

```
트랜잭션 보장 체계

원자성 + 영속성 → 복구 시스템 (Chapter 19)
고립성          → 동시성 제어 (Chapter 18)
일관성          → 응용 프로그래머 책임

직렬가능성:
  충돌 직렬가능 > 뷰 직렬가능
  선행 그래프에 사이클 없음 ↔ 충돌 직렬가능

스케줄 안전성:
  연쇄 없는 → 회복 가능
  회복 가능 ⊆ 모든 스케줄
```

| 개념 | 핵심 포인트 |
|------|------------|
| ACID | Atomicity(복구), Consistency(프로그래머), Isolation(동시성제어), Durability(복구) |
| 직렬 스케줄 | n개 트랜잭션 → n! 가지 직렬 스케줄 |
| 충돌 직렬가능 | 선행 그래프에 사이클 없음 ↔ 충돌 직렬가능 |
| 회복 가능 | 종속 트랜잭션 커밋 순서 보장 |
| 연쇄 없음 | 커밋된 데이터만 읽기 (uncommitted read 금지) |
| Snapshot Isolation | 읽기 블로킹 없음, Write Skew 가능 |
| 팬텀 현상 | 술어 충돌, 인덱스 잠금으로 해결 |
