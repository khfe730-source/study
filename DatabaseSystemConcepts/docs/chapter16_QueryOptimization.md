# Chapter 16: Query Optimization

## 개요

쿼리 최적화는 주어진 쿼리와 동치인 실행 계획들 중 비용이 가장 낮은 계획을 선택하는 과정이다. 동치 변환 규칙(16 가지)으로 대안 식을 생성하고, 카탈로그 통계·히스토그램으로 결과 크기를 추정하며, 동적 프로그래밍으로 최적 조인 순서를 결정한다. 실체화 뷰 유지·쿼리 재작성, 상관 서브쿼리의 세미조인으로의 탈상관화(decorrelation), 고급 최적화(Top-K, Join Minimization, Halloween Problem, Multi-Query)를 다룬다.

---

## 16.1 개요

```
SQL 쿼리 (쿼리 1)
    │
    ▼ 관계 대수 표현 1
Πname,title(σdept_name="Music"(instructor ⋈ (teaches ⋈ Πcourse_id,title(course))))
    │
    ▼ 동치 변환 규칙 적용 (선택 조기 처리)
Πname,title((σdept_name="Music"(instructor)) ⋈ (teaches ⋈ Πcourse_id,title(course)))
    │
    ▼ 알고리즘 선택 + 파이프라이닝 명시
실행 계획 (Evaluation Plan):
  - σdept_name="Music": index 1 사용
  - instructor ⋈ teaches: sort + merge join (ID 기준 정렬)
  - 중간결과 ⋈ course: hash join
  - 최종 Π: sort (중복 제거)
```

**쿼리 평가 계획 확인** (실무):
```sql
-- PostgreSQL / MySQL
EXPLAIN SELECT ...;

-- Oracle
EXPLAIN PLAN FOR SELECT ...;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- SQL Server
SET SHOWPLAN_TEXT ON;
SELECT ...;
```

---

## 16.2 관계 대수 식의 동치 변환

두 식이 동치(Equivalent) ↔ 모든 유효 DB 인스턴스에서 동일한 튜플 집합 생성.

### 핵심 동치 변환 규칙 (16가지)

| # | 규칙 | 설명 |
|---|------|------|
| 1 | σθ1∧θ2(E) ≡ σθ1(σθ2(E)) | 선택의 연쇄 분해 |
| 2 | σθ1(σθ2(E)) ≡ σθ2(σθ1(E)) | 선택 교환 가능 |
| 3 | ΠL1(ΠL2(…(ΠLn(E))…)) ≡ ΠL1(E) | 투영 연쇄 단순화 |
| 4a | σθ(E1 × E2) ≡ E1 ⋈θ E2 | θ-조인 정의 |
| 4b | σθ1(E1 ⋈θ2 E2) ≡ E1 ⋈θ1∧θ2 E2 | 선택과 θ-조인 결합 |
| 5 | E1 ⋈θ E2 ≡ E2 ⋈θ E1 | 조인 교환 법칙 |
| 6a | (E1 ⋈ E2) ⋈ E3 ≡ E1 ⋈ (E2 ⋈ E3) | 자연 조인 결합 법칙 |
| 6b | (E1 ⋈θ1 E2) ⋈θ2∧θ3 E3 ≡ E1 ⋈θ1∧θ3 (E2 ⋈θ2 E3) | θ-조인 결합 법칙 |
| 7a | σθ1(E1 ⋈θ E2) ≡ (σθ1(E1)) ⋈θ E2  (θ1은 E1 속성만) | 선택의 조인 분배 |
| 7b | σθ1∧θ2(E1 ⋈θ E2) ≡ (σθ1(E1)) ⋈θ (σθ2(E2)) | 선택 분배 (양쪽 독립) |
| 8a | ΠL1∪L2(E1 ⋈θ E2) ≡ (ΠL1(E1)) ⋈θ (ΠL2(E2))  (θ는 L1∪L2만 사용) | 투영 조인 분배 |
| 9 | E1 ∪ E2 ≡ E2 ∪ E1, E1 ∩ E2 ≡ E2 ∩ E1 | 합집합·교집합 교환 |
| 11 | σθ(E1 ∪ E2) ≡ σθ(E1) ∪ σθ(E2) | 선택의 집합 연산 분배 |
| 13 | σθ(GγA(E)) ≡ GγA(σθ(E))  (θ는 G만 사용) | 선택과 집계 교환 |
| 15a | σθ1(E1 ⟕θ E2) ≡ (σθ1(E1)) ⟕θ E2  (θ1은 E1 속성만) | 선택의 외부 조인 분배 |
| 16 | σθ1(E1 ⟕θ E2) ≡ σθ1(E1 ⋈θ E2)  (θ1이 E2 NULL 거부) | 외부조인 → 내부조인 변환 |

**주의 사항**:
- 외부 조인은 결합 법칙 성립 안 함: `(r ⟕ s) ⟕ t ≢ r ⟕ (s ⟕ t)`
- 선택은 특정 조건 없이는 외부 조인에 대해 분배되지 않음.
- 규칙 16: NULL을 거부하는 선택이 있으면 Left Outer Join → Inner Join으로 대체 가능.

### 변환 예시

```
원래: Πname,title(σdept_name="Music"∧year=2017(instructor ⋈ (teaches ⋈ Πcourse_id,title(course))))

1. 규칙 6a (결합법칙):
   → σdept_name="Music"∧year=2017((instructor ⋈ teaches) ⋈ Πcourse_id,title(course))

2. 규칙 7a (선택 분배):
   → (σdept_name="Music"∧year=2017(instructor ⋈ teaches)) ⋈ Πcourse_id,title(course)

3. 규칙 1 (선택 분해):
   → (σdept_name="Music"(σyear=2017(instructor ⋈ teaches))) ⋈ Πcourse_id,title(course)

4. 규칙 7b (선택 양쪽 분배):
   → (σdept_name="Music"(instructor) ⋈ σyear=2017(teaches)) ⋈ Πcourse_id,title(course)
   ← 이것이 최적에 가까운 형태 (선택을 조기 처리!)
```

### 조인 순서의 중요성

```
n개 릴레이션 조인 가능한 순서 수: (2(n-1))! / (n-1)!
  n=3:  12가지
  n=5:  1,680가지
  n=7:  665,280가지
  n=10: 17.6 billion가지!

잘못된 순서 예:
instructor × Πcourse_id,title(course) ⋈ teaches
  → 카르테시안 곱이 먼저 발생 (매우 큰 중간 결과)

올바른 순서:
(instructor ⋈ teaches) ⋈ Πcourse_id,title(course)
  → instructor ⋈ teaches = 각 교수가 가르친 수업 (크기 적절)
```

---

## 16.3 결과 크기 통계 추정

### 16.3.1 카탈로그 정보

| 통계 | 설명 |
|------|------|
| nr | 릴레이션 r의 튜플 수 |
| br | r의 블록 수 |
| lr | 튜플 크기 (bytes) |
| fr | 블록당 튜플 수 (Blocking Factor) |
| V(A, r) | 속성 A의 고유값(distinct value) 수 |

**히스토그램**:
- 등폭(Equi-Width): 값 범위를 동일 크기 구간으로 분할
- 등깊이(Equi-Depth): 각 구간마다 동일한 튜플 수 → 더 정확하고 공간 효율적

```
속성 age (0~99) 등깊이 히스토그램:
경계: (4, 8, 14, 19)
→ 각 구간에 1/5 튜플 분포
```

**빈도 높은 값 별도 저장**: 상위 n개 값의 정확한 빈도를 카탈로그에 저장. 나머지 값은 히스토그램 기반.

### 16.3.2 셀렉션 크기 추정

**σA=a(r)**:
- 균등 분포 가정: 결과 크기 ≈ nr / V(A, r)
- 히스토그램 있으면: 해당 구간의 빈도 / 구간 내 고유값 수

**σA≤v(r)**:
```
결과 크기 ≈ nr × (v - min(A,r)) / (max(A,r) - min(A,r))
           0  if v < min(A,r)
           nr if v ≥ max(A,r)
```

**복합 선택**:
- **합집합(AND)**: 각 θi의 선택도 si/nr → 결과 크기 ≈ nr × (s1×s2×…×sn) / nr^n
- **합집합(OR)**: 결과 크기 ≈ nr × [1 − (1-s1/nr)(1-s2/nr)…(1-sn/nr)]
- **부정(NOT)**: nr − |σθ(r)|

### 16.3.3 조인 크기 추정

**r ⋈ s (R∩S = {A})**:

```
경우 1: R∩S = ∅ → 카르테시안 곱, nr × ns
경우 2: R∩S가 R의 키 → 결과 ≤ ns
경우 3: A가 S의 외래키(referencing R) → 결과 = ns
경우 4: 일반 경우:
  추정값 = min(nr×ns / V(A,s),  nr×ns / V(A,r))
```

**예**: student ⋈ takes, V(ID, takes)=2500, V(ID, student)=5000:
- 추정 1: 5000×10000/2500 = 20,000
- 추정 2: 5000×10000/5000 = 10,000 ← 낮은 값 선택
- 실제(외래키 알면): ntakes = 10,000 ✓

### 16.3.4 기타 연산 크기 추정

| 연산 | 크기 추정 |
|------|---------|
| ΠA(r) | V(A, r) |
| GγA(r) | V(G, r) |
| r ∪ s | |r| + |s| (상한) |
| r ∩ s | min(|r|, |s|) (상한) |
| r − s | |r| (상한) |
| r ⟕ s | |r ⋈ s| + |r| (상한) |
| r ⟗ s | |r ⋈ s| + |r| + |s| (상한) |

---

## 16.4 최적 실행 계획 선택

### 16.4.1 동적 프로그래밍 기반 조인 순서 최적화

**핵심 아이디어**: 부분 집합 S에 대한 최적 계획을 구하면, 그 결과를 다른 릴레이션과 조인할 때 재활용 가능. 중복 계산 방지.

```python
procedure FindBestPlan(S):
  if bestplan[S].cost ≠ ∞:  return bestplan[S]  # 메모이제이션
  if |S| == 1:
    S 단일 릴레이션에 대한 최적 접근법 (인덱스 스캔 or 전체 스캔)
  else:
    for each non-empty subset S1 ⊂ S (S1 ≠ S):
      P1 = FindBestPlan(S1)
      P2 = FindBestPlan(S − S1)
      for each join algorithm A for P1 ⋈ P2:
        cost = P1.cost + P2.cost + cost(A)
        if cost < bestplan[S].cost:
          bestplan[S] = (plan, cost)
  return bestplan[S]
```

**시간 복잡도**: O(3ⁿ). n=10이면 약 59,000 계산 (17.6B vs 59K).

**흥미로운 정렬 순서(Interesting Sort Order)**: 이후 조인에서 활용 가능한 정렬 순서 보존. 병합 조인 선택 시 비용이 조금 높아도 정렬된 결과를 유지하면 전체 비용 감소 가능.

bestplan을 [S, 정렬순서]로 인덱스하여 확장.

### 16.4.2 동치 규칙 기반 범용 최적화기 (Volcano/Cascades)

물리 동치 규칙(Physical Equivalence Rules) 추가:
```
논리 연산: 조인 → 물리 연산: 해시 조인, 병합 조인, 블록 중첩 루프 조인 ...
```

효율화 기법:
1. 서브식 공유 표현 (공유 포인터)
2. 중복 파생 식 탐지
3. 메모이제이션 (동적 프로그래밍)
4. 가지 치기 (현재까지 최저 비용보다 비싼 계획 제거)

SQL Server 옵티마이저가 이 접근법 기반.

### 16.4.3 최적화 휴리스틱

**선택 조기 처리 (Perform Selection Early)**:
- 가장 강력한 휴리스틱 → 중간 결과 크기 대폭 감소
- 예외: r이 매우 작고 s에 인덱스가 있을 때, σθ를 s에 먼저 적용하면 인덱스 활용 못할 수 있음

**투영 조기 처리 (Perform Projection Early)**:
- 불필요한 속성 조기 제거로 중간 릴레이션 크기 감소

**왼쪽-깊은 조인 트리 (Left-Deep Join Tree)**:
```
         ⋈
        / \
       ⋈   r5
      / \
     ⋈   r4
    / \
   ⋈   r3
  / \
 r1  r2

특징: 오른쪽 피연산자는 항상 저장된 릴레이션
      파이프라인 평가에 적합
System R 옵티마이저가 이 형태만 고려 → O(n!) 대신 O(n·2ⁿ)
```

**플랜 캐싱 (Plan Caching)**:
- 파라미터 값이 다르더라도 첫 실행 때의 계획 재사용 (빠른 최적화)
- 최적 계획이 파라미터 값에 따라 크게 달라지면 비효율

### 16.4.4 중첩 서브쿼리 최적화 (Decorrelation)

**상관 평가(Correlated Evaluation)**: 외부 쿼리 각 튜플마다 서브쿼리 실행 → 비효율적.

```sql
-- 상관 서브쿼리 예
SELECT name FROM instructor
WHERE EXISTS (
    SELECT * FROM teaches
    WHERE instructor.ID = teaches.ID AND teaches.year = 2019
);
```

**탈상관화(Decorrelation)**: 세미조인으로 변환.

```
세미조인 (⋉θ): ri가 r에 n번 등장하면, si ∈ s중 ri와 θ 만족하는 것이 있으면 ri를 n번 결과에 포함.

EXISTS 변환:
  Πname(instructor ⋉(instructor.ID=teaches.ID∧year=2019) teaches)

NOT EXISTS 변환 (Anti-semijoin ⋉̄):
  Πname(instructor ⋉̄ (instructor.ID=teaches.ID) (σyear=2019(teaches)))
```

**집계 포함 서브쿼리 탈상관화**:
```sql
SELECT name FROM instructor
WHERE 1 < (SELECT count(*) FROM teaches
           WHERE instructor.ID = teaches.ID AND year = 2019);
```
→
```
Πname(instructor ⋉(instructor.ID=TID)∧(1<cnt) (ID as TID γcount(*) as cnt (σyear=2019(teaches))))
```
서브쿼리의 GROUP BY 없던 집계가 ID 기준 GROUP BY로 변환됨.

---

## 16.5 실체화 뷰 (Materialized Views)

### 16.5.1 뷰 유지 (View Maintenance)

```sql
CREATE VIEW dept_total_salary(dept_name, total_salary) AS
SELECT dept_name, SUM(salary) FROM instructor GROUP BY dept_name;
```

**재계산 방식**: 뷰를 통째로 재계산 → 비용 높음.

**점진적 뷰 유지(Incremental View Maintenance)**: 변경된 부분만 업데이트.

- **즉시 뷰 유지(Immediate)**: 업데이트 트랜잭션과 함께 즉시 반영.
- **지연 뷰 유지(Deferred)**: 나중에 일괄 처리 (일관성 불보장 가능).

### 16.5.2 점진적 유지 방법

**차분(Differential)**: ir = r에 삽입된 튜플 집합, dr = 삭제된 튜플 집합.

**조인(v = r ⋈ s)**:
```
ir 삽입 시: vnew = vold ∪ (ir ⋈ s)
dr 삭제 시: vnew = vold − (dr ⋈ s)
```

**셀렉션(v = σθ(r))**:
```
ir 삽입 시: vnew = vold ∪ σθ(ir)
dr 삭제 시: vnew = vold − σθ(dr)
```

**투영(v = ΠA(r))**: 레코드마다 카운트 유지 필요.
```
삽입: (t.A)가 이미 존재하면 카운트+1, 없으면 (t.A, 카운트=1) 추가
삭제: 카운트−1, 0이 되면 제거
```

**집계**:
| 집계 함수 | 삽입 처리 | 삭제 처리 |
|---------|---------|---------|
| count | 그룹 존재 시 +1, 없으면 (G, 1) 추가 | −1, 0이면 그룹 삭제 |
| sum | sum 누적, count 유지 | sum−t.B, count−1 |
| avg | sum+count 유지 후 avg=sum/count | 위와 동일 |
| min/max | 삽입 쉬움 | 삭제 후 다음 min/max 탐색 필요 (인덱스 권장) |

### 16.5.3 실체화 뷰와 쿼리 최적화

**뷰를 이용한 쿼리 재작성**:
```
v = r ⋈ s가 실체화되어 있고, 사용자가 r ⋈ s ⋈ t 쿼리 제출 시:
  → v ⋈ t로 재작성 → 비용 절감
```

**뷰 정의로 대체**:
```
v = r ⋈ s, 인덱스 없음, 사용자가 σA=10(v) 제출 시:
  → σA=10(r) ⋈ s로 재작성 + r.A, s.B 인덱스 활용 → 더 효율적일 수 있음
```

**뷰 선택(Materialized View Selection)**: 워크로드 기반으로 어떤 뷰를 실체화할지 결정. Microsoft SQL Server Database Tuning Assistant, IBM DB2 Design Advisor 등 도구 제공.

---

## 16.6 고급 최적화 기법

### 16.6.1 Top-K 최적화

`LIMIT K` 쿼리에서 전체 결과 생성 후 K개 선택은 비효율. 파이프라인 기반 계획으로 정렬된 순서로 결과 생성, 또는 선택 조건 추가하여 상위 K개 예상 범위의 값만 처리.

### 16.6.2 조인 최소화 (Join Minimization)

뷰를 통해 불필요한 조인이 포함된 경우:
```sql
-- v = instructor ⋈ department 뷰 사용
-- 쿼리에서 department 속성을 전혀 사용 안 함
-- + instructor.dept_name NOT NULL (FK) 인 경우

-- department와의 조인 제거 가능 (결과에 영향 없음)
```

### 16.6.3 업데이트 최적화 (Halloween Problem)

```sql
UPDATE instructor SET salary = salary * 1.1
WHERE salary >= 100000;
```

**Halloween Problem**: 인덱스 스캔 중 업데이트된 튜플이 인덱스 앞쪽에 재삽입 → 같은 튜플이 여러 번 업데이트될 수 있음.

**해결책**: 업데이트 조건 평가와 실제 업데이트를 분리 (먼저 영향 받는 튜플 목록 수집 후 업데이트).

인덱스 속성에 영향 없는 업데이트, 또는 인덱스 스캔 방향과 반대 방향으로 값 변경 시에는 문제 없음.

**배치 업데이트 최적화**: 대량 업데이트를 모아서 인덱스 순서로 정렬 후 일괄 처리 → 랜덤 I/O 감소.

### 16.6.4 다중 쿼리 최적화 (Multiquery Optimization)

여러 쿼리 동시 제출 시 공통 서브식을 한 번만 계산하여 재활용.

**공통 서브식 제거(Common Subexpression Elimination)**: 동일 서브식이 여러 쿼리에 등장하면 한 번만 계산하여 공유.

**공유 스캔(Shared Scan)**: 동일 대형 릴레이션을 여러 쿼리가 동시에 스캔할 때, 한 번의 디스크 읽기로 여러 쿼리에 파이프라인 전달.

### 16.6.5 파라미터 쿼리 최적화 (Parametric Query Optimization)

파라미터 값 없이 최적화 수행 → 파라미터 값 범위별로 여러 계획 저장. 쿼리 실행 시 파라미터 값에 맞는 계획 선택 (전체 최적화보다 빠름).

### 16.6.6 적응형 쿼리 처리 (Adaptive Query Processing)

실행 중 수집된 통계가 옵티마이저 추정과 크게 다를 경우 계획 변경.

- SQL Server의 적응형 조인: 외부 입력 크기 확인 후 중첩 루프 조인 vs 해시 조인 동적 결정.
- 실행 중 통계 기반 계획 재최적화 (abort → 재실행 방지 전략 포함).

---

## 16.7 핵심 요약

```
쿼리 최적화 파이프라인

SQL →[파싱]→ 관계 대수
    →[동치 변환]→ 대안 식 생성
         ↑ 16가지 동치 규칙
    →[통계 추정]→ 각 식의 비용 계산
         ↑ nr, V(A,r), 히스토그램
    →[계획 선택]→ 최저 비용 계획 선택
         ↑ 동적 프로그래밍 (조인 순서)
         ↑ 휴리스틱 (선택/투영 조기 처리)
    →[탈상관화]→ 상관 서브쿼리 → 세미조인
    →[실체화 뷰]→ 점진적 유지 + 쿼리 재작성
```

| 개념 | 핵심 포인트 |
|------|------------|
| 동치 규칙 | 16가지, 조인 교환·결합, 선택 조기 처리가 핵심 |
| 선택 조기 처리 | 중간 결과 크기 감소 → 대부분 비용 절감 |
| 크기 추정 | nr/V(A,r), 히스토그램, 조인: min(nr×ns/V(A,s), nr×ns/V(A,r)) |
| 동적 프로그래밍 | O(3ⁿ), n=10에서 59K 계산, 부분집합별 최적 계획 저장 |
| 왼쪽-깊은 트리 | 오른쪽은 항상 저장 릴레이션, 파이프라인 적합 |
| 탈상관화 | EXISTS → 세미조인(⋉), NOT EXISTS → 안티세미조인(⋉̄) |
| 점진적 뷰 유지 | 삽입/삭제 차분으로 뷰만 업데이트 (전체 재계산 불필요) |
| Halloween Problem | 업데이트 + 인덱스 스캔 동시 → 이중 적용 위험 |
| 플랜 캐싱 | 파라미터 달라도 첫 계획 재사용 (빠르지만 부정확 가능) |
| 적응형 처리 | 실행 중 통계 수집 후 계획 동적 변경 |
