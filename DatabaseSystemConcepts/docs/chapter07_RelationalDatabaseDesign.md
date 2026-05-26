# Chapter 7: Relational Database Design

## 개요

함수 종속성(functional dependency)을 형식적 도구로 삼아 관계형 데이터베이스 스키마의 "좋음"을 정의하고, 불필요한 중복을 제거하는 정규화(normalization) 이론을 다룬다. BCNF와 3NF를 중심으로, 분해 알고리즘과 그 정확성까지 상세히 설명한다.

---

## 7.1 좋은 관계형 설계의 특성 (Features of Good Relational Designs)

### 7.1.1 문제: 스키마 합병의 부작용

Chapter 6에서 `instructor`와 `department`를 별개 릴레이션으로 관리했지만, 이를 합쳐 아래처럼 설계하면 어떻게 될까?

```
in_dep(ID, name, salary, dept_name, building, budget)
```

**발생하는 문제**:

1. **정보 중복(Redundancy)**: `dept_name` → `building, budget` 이므로, 동일 부서의 instructor가 여러 명이면 building/budget이 중복 저장됨
2. **갱신 이상(Update Anomaly)**: budget을 수정할 때 모든 튜플을 갱신해야 → 일관성 위반 가능
3. **삽입 이상(Insertion Anomaly)**: 교수가 없는 신규 부서는 표현 불가 (ID, name, salary에 null 필요)

**결론**: `in_dep`는 나쁜 설계. `instructor`와 `department`로 분리해야 한다.

### 7.1.2 무손실 분해 (Lossless Decomposition)

분해가 무손실인 조건:

```
R = R1 ∪ R2 이고, 다음이 성립하면 무손실:
ΠR₁(r) ⋈ ΠR₂(r) = r
```

**손실 분해(Lossy Decomposition) 예**:

```
employee(ID, name, street, city, salary)
→ employee1(ID, name)
→ employee2(name, street, city, salary)
```

동명이인이 있으면 자연 조인 결과가 원본보다 커진다:
```
원본: (57766, Kim, Main, Perryridge, 75000)
      (98776, Kim, North, Hampton,   67000)

employee1 ⋈ employee2 결과:
  (57766, Kim, Main, Perryridge, 75000)
  (57766, Kim, North, Hampton, 67000)   ← 잘못된 튜플
  (98776, Kim, Main, Perryridge, 75000) ← 잘못된 튜플
  (98776, Kim, North, Hampton, 67000)
```

더 많은 튜플을 갖지만 실제로 더 적은 정보 (누가 어느 주소와 연결되는지 알 수 없음).

**무손실 분해를 위한 함수 종속성 조건**:

```
R1 ∩ R2 → R1   또는   R1 ∩ R2 → R2
```

즉, 교집합이 R1 또는 R2의 슈퍼키이면 무손실.

```
예: in_dep → instructor(ID, name, dept_name, salary)
            + department(dept_name, building, budget)
교집합: {dept_name}
dept_name → building, budget 이므로 dept_name은 department의 슈퍼키 → 무손실 ✓
```

---

## 7.2 함수 종속성을 이용한 분해 (Decomposition Using Functional Dependencies)

### 7.2.1 표기 규약

- `α, β, γ, …`: 속성 집합 (set of attributes)
- 대문자 로마자(`R`, `S`, …): 릴레이션 스키마
- `r(R)`: 스키마 R의 릴레이션 r
- `K`: 슈퍼키
- `αβ`: α ∪ β의 축약 표기

### 7.2.2 함수 종속성 (Functional Dependencies)

**정의**: 릴레이션 스키마 r(R)에서 α ⊆ R, β ⊆ R에 대해:

> α → β (α는 β를 함수적으로 결정한다)
>
> ↔ 모든 합법적 인스턴스에서, t₁[α] = t₂[α] 이면 t₁[β] = t₂[β]

- **슈퍼키**: K → R을 만족하는 K
- **합법적 인스턴스(Legal Instance)**: 실세계 제약을 모두 만족하는 릴레이션 인스턴스

**FD의 두 가지 용도**:
1. 주어진 인스턴스가 FD를 만족하는지 **검증**
2. 합법적 릴레이션의 집합을 **제약** (스키마 설계 도구)

**자명한 FD (Trivial FD)**: β ⊆ α이면 α → β는 항상 성립 (모든 릴레이션에서 자동으로 참)

```
예: AB → A, AB → B, A → A  ← 모두 trivial
```

**F⁺**: FD 집합 F의 클로저 (F로부터 논리적으로 도출 가능한 모든 FD의 집합)

---

## 7.3 정규형 (Normal Forms)

### 7.3.1 보이스-코드 정규형 (BCNF, Boyce-Codd Normal Form)

**정의**: 릴레이션 스키마 R이 FD 집합 F에 대해 BCNF이려면, F⁺의 모든 비자명 FD `α → β`에 대해:

```
1) α → β가 trivial (β ⊆ α)   OR
2) α가 R의 슈퍼키
```

두 조건 중 하나를 반드시 만족해야 한다.

**BCNF 위반 예**: `in_dep(ID, name, salary, dept_name, building, budget)`
- `dept_name → building, budget` 성립
- 그러나 `dept_name`은 `in_dep`의 슈퍼키가 아님 → **BCNF 위반**

**BCNF 분해 규칙**: α → β가 BCNF를 위반할 때 (α가 슈퍼키가 아닌 비자명 FD), R을:
```
(α ∪ β)        와    (R − (β − α))
```
으로 분해.

```
예: in_dep, α = dept_name, β = {building, budget}
→ (dept_name, building, budget)  = department
→ (ID, name, dept_name, salary)  = instructor
```

#### BCNF와 의존성 보존 (Dependency Preservation)

**문제**: BCNF 분해가 항상 의존성을 보존하지는 않는다.

```
dept_advisor(s_ID, i_ID, dept_name)

FDs:
f1: i_ID → dept_name        (교수는 하나의 부서에만 어드바이저)
f2: s_ID, dept_name → i_ID  (학생은 부서당 최대 하나의 어드바이저)

i_ID는 슈퍼키가 아님 → BCNF 위반

BCNF 분해:
(s_ID, i_ID)
(i_ID, dept_name)
```

분해 후 `f2: s_ID, dept_name → i_ID`는 어느 단일 릴레이션에도 포함되지 않음 → **의존성 비보존**.

이 FD를 검증하려면 매번 조인이 필요 → 비용 증가.

### 7.3.2 제3 정규형 (3NF, Third Normal Form)

**정의**: BCNF의 조건을 약간 완화. F⁺의 모든 비자명 FD `α → β`에 대해:

```
1) α → β가 trivial            OR
2) α가 R의 슈퍼키             OR
3) β − α의 각 속성 A가 R의 어떤 후보키에 포함됨
```

**직관**: 3번 조건은 "후보키의 일부로 가는 종속성"은 허용. 이 완화 덕분에 의존성 보존 분해가 항상 가능.

```
dept_advisor 재검토:
f1: i_ID → dept_name
  - i_ID는 슈퍼키 아님
  - dept_name ∈ β−α = {dept_name}
  - f2: s_ID, dept_name → i_ID 이므로 dept_name은 후보키 {s_ID, dept_name}에 포함
  → 3NF 조건 3번 만족 → dept_advisor는 3NF ✓ (BCNF는 아님)
```

### 7.3.3 BCNF vs. 3NF 비교

| 항목 | BCNF | 3NF |
|------|------|-----|
| 중복 제거 | 완전 (FD 기반) | 완전하지 않을 수 있음 |
| 무손실 분해 | 항상 가능 | 항상 가능 |
| 의존성 보존 | 불가능할 수 있음 | **항상 가능** |
| null 필요성 | 적음 | 있을 수 있음 |

**설계 목표 우선순위**:
1. BCNF
2. 무손실성(Losslessness)
3. 의존성 보존(Dependency Preservation)

세 가지를 모두 만족할 수 없을 때: BCNF 포기 + 3NF 선택.

**SQL과의 관계**: SQL은 `PRIMARY KEY`, `UNIQUE` 제약으로만 FD를 효율적으로 검사 가능. 임의 FD는 assertion으로 표현 가능하나 현실적으로 지원되지 않음.

### 7.3.4 더 높은 정규형

FD만으로는 제거할 수 없는 중복이 존재:

```
(ID, child_name, phone_number)
→ 교수 ID 99999, 자녀 David/William, 전화 512-1234/512-4321

필요한 튜플:
(99999, David,   512-1234)
(99999, David,   512-4321)
(99999, William, 512-1234)
(99999, William, 512-4321)
```

이 스키마는 BCNF이지만 전화번호와 자녀 정보가 독립적 관계임에도 중복 발생.
→ 다중값 종속성(MVD)과 4NF로 해결.

---

## 7.4 함수 종속성 이론 (Functional-Dependency Theory)

### 7.4.1 F의 클로저 F⁺ 와 Armstrong의 공리

**F를 이용한 새로운 FD 도출**: Armstrong의 공리(Armstrong's Axioms) 사용

| 공리 | 내용 | 직관 |
|------|------|------|
| **반사율(Reflexivity)** | β ⊆ α이면 α → β | trivial FD |
| **확장율(Augmentation)** | α → β이면 γα → γβ | 양쪽에 같은 속성 추가 |
| **이행율(Transitivity)** | α → β, β → γ이면 α → γ | 연결 |

이 세 공리는 **완전(Complete)**하고 **건전(Sound)**: F⁺의 모든 FD를 도출 가능, 도출된 것은 모두 올바름.

**추가 유도 규칙** (Armstrong 공리로 증명 가능):

| 규칙 | 내용 |
|------|------|
| **합집합(Union)** | α→β, α→γ이면 α→βγ |
| **분해(Decomposition)** | α→βγ이면 α→β, α→γ |
| **유사이행(Pseudotransitivity)** | α→β, γβ→δ이면 αγ→δ |

**F⁺ 계산 예**:
```
R = (A, B, C, G, H, I)
F = {A→B, A→C, CG→H, CG→I, B→H}

도출:
A→H   (A→B, B→H, 이행율)
CG→HI (CG→H, CG→I, 합집합)
AG→I  (A→C, CG→I, 유사이행율)
```

### 7.4.2 속성 클로저 (Attribute Closure) α⁺

**α⁺ 정의**: FD 집합 F 하에서 α가 함수적으로 결정하는 모든 속성의 집합

**알고리즘** (α⁺ 계산):

```python
result = α
repeat:
    for each FD β → γ in F:
        if β ⊆ result:
            result = result ∪ γ
until result does not change
return result
```

**예**: F = {A→B, A→C, CG→H, CG→I, B→H}, α = AG
```
초기: result = {A, G}
A→B:    result = {A, B, G}
A→C:    result = {A, B, C, G}
CG→H:   result = {A, B, C, G, H}
CG→I:   result = {A, B, C, G, H, I}
(AG)⁺ = {A, B, C, G, H, I} = R → AG는 슈퍼키
```

**α⁺의 활용**:
1. **슈퍼키 테스트**: α⁺ = R이면 α는 슈퍼키
2. **FD 검증**: α→β가 F⁺에 속하는지 ← β ⊆ α⁺ 확인
3. **F⁺ 계산**: 각 γ ⊆ R에 대해 γ⁺ 계산, S ⊆ γ⁺인 모든 S에 대해 γ→S 생성

### 7.4.3 정준 커버 (Canonical Cover) Fc

**목적**: FD 집합을 검사 비용 최소화를 위해 동등한 최소 집합으로 축약.

**불필요 속성(Extraneous Attribute)**: 제거해도 F의 클로저가 변하지 않는 속성

- **우변에서 제거**: A ∈ β → F' = (F − {α→β}) ∪ {α→(β−A)} 하에서 α⁺를 계산, A ∈ α⁺이면 A는 불필요
- **좌변에서 제거**: A ∈ α → γ = α−{A}, F 하에서 γ⁺를 계산, β ⊆ γ⁺이면 A는 불필요

**정준 커버 Fc의 조건**:
1. Fc의 어떤 FD에도 불필요 속성이 없음
2. Fc의 각 FD는 유일한 좌변 (α₁→β₁과 α₁→β₂가 동시에 존재하면 α₁→β₁β₂로 합침)
3. F⁺ = Fc⁺

**Fc 계산 알고리즘**:

```
Fc = F
repeat:
    동일 좌변의 FD들을 합집합 규칙으로 합침 (α→β₁, α→β₂ → α→β₁β₂)
    불필요 속성을 찾아서 제거
until Fc 변화 없음
```

**예**: F = {A→BC, B→C, A→B, AB→C}

```
1. A→BC와 A→B를 합침: A→BC (이미 BC 포함)
   → F = {A→BC, B→C, AB→C}
2. AB→C에서 A가 불필요: B→C가 있으므로 B→C로 충분
   → F = {A→BC, B→C}
3. A→BC에서 C가 불필요: A→B, B→C로 A→C 도출 가능
   → Fc = {A→B, B→C}
```

### 7.4.4 의존성 보존 (Dependency Preservation)

분해 D = {R1, R2, …, Rn}이 F에 대해 의존성 보존인 조건:

```
Fi = F⁺의 Ri 제한 (Ri 속성만 포함하는 FD들)
F' = F1 ∪ F2 ∪ … ∪ Fn
F'⁺ = F⁺
```

**효율적 검사** (지수 시간의 F⁺ 계산 없이, 다항 시간):

각 FD `α → β` in F에 대해:
```
result = α
repeat:
    for each Ri in D:
        t = (result ∩ Ri)⁺ ∩ Ri   // F를 사용한 클로저
        result = result ∪ t
until result 변화 없음
if β ⊆ result: 이 FD는 보존됨
```

---

## 7.5 함수 종속성을 이용한 분해 알고리즘

### 7.5.1 BCNF 분해 알고리즘

#### BCNF 검사

릴레이션 스키마 R과 FD 집합 F에 대해:
1. F의 각 FD `α → β`에 대해 α⁺를 계산
2. α⁺가 R 전체를 포함하지 않으면 BCNF 위반

> ⚠️ 분해 후 서브스키마 Ri에 대해서는 F 대신 F⁺의 제한(restriction)을 써야 함. F만으로는 누락된 위반을 찾지 못할 수 있음.

#### BCNF 분해 알고리즘

```
result = {R}
done = false
while not done:
    if 어떤 Ri ∈ result이 BCNF가 아님:
        α → β를 BCNF 위반하는 비자명 FD로 선택
        (단, α ∩ β = ∅ 조건 적용)
        result = (result − {Ri}) ∪ {Ri − β} ∪ {α, β}
    else:
        done = true
```

**무손실 보장**: 분해 시 (Ri − β) ∩ (α, β) = α이고 α → β가 성립 → 무손실 조건 만족.

**시간 복잡도**: 지수 시간 (worst case). 다항 시간 알고리즘도 존재하나 과도 정규화(overnormalization) 가능성 있음.

#### BCNF 분해 예시: `class` 스키마

```
class(course_id, title, dept_name, credits, sec_id, semester, year,
      building, room_number, capacity, time_slot_id)

FDs:
f1: course_id → title, dept_name, credits
f2: building, room_number → capacity
f3: course_id, sec_id, semester, year → building, room_number, time_slot_id

후보키: {course_id, sec_id, semester, year}
```

**Step 1**: f1 위반 (course_id가 슈퍼키 아님)
```
→ course(course_id, title, dept_name, credits)  [BCNF ✓]
→ class-1(course_id, sec_id, semester, year, building, room_number, capacity, time_slot_id)
```

**Step 2**: f2 위반 ({building, room_number}이 class-1의 슈퍼키 아님)
```
→ classroom(building, room_number, capacity)  [BCNF ✓]
→ section(course_id, sec_id, semester, year, building, room_number, time_slot_id)  [BCNF ✓]
```

최종 결과: `course`, `classroom`, `section` — 무손실 + 의존성 보존 ✓

### 7.5.2 3NF 분해 알고리즘 (합성 알고리즘, Synthesis Algorithm)

```
1. Fc = F의 정준 커버 계산
2. i = 0
3. Fc의 각 FD α → β에 대해:
     i += 1
     Ri = αβ   (스키마 추가)
4. 어떤 Rj도 R의 후보키를 포함하지 않으면:
     i += 1
     Ri = R의 임의 후보키   (무손실 보장)
5. 다른 스키마에 포함된 스키마 제거 (Rj ⊆ Rk이면 Rj 삭제)
6. {R1, R2, ..., Ri} 반환
```

**보장**:
- 의존성 보존: Fc의 각 FD에 대해 명시적으로 스키마 생성
- 무손실: 후보키를 포함하는 스키마가 반드시 존재

**시간 복잡도**: 다항 시간 (BCNF 검사는 NP-hard이지만 3NF 합성은 polynomial)

**예시**: `dept_advisor(s_ID, i_ID, dept_name)`

Fc = {i_ID → dept_name, s_ID dept_name → i_ID}

```
Step 3: R1 = (i_ID, dept_name)
        R2 = (s_ID, dept_name, i_ID)
Step 4: R2는 후보키 {s_ID, dept_name}을 포함 → 추가 스키마 불필요
Step 5: R1 ⊆ R2이므로 R1 삭제
최종: dept_advisor(s_ID, i_ID, dept_name)  [그대로]
```

---

## 7.6 다중값 종속성을 이용한 분해 (Decomposition Using Multivalued Dependencies)

### 7.6.1 다중값 종속성 (Multivalued Dependency, MVD)

**동기**: BCNF를 만족하면서도 중복이 있는 경우:

```
r2(ID, dept_name, street, city)
FD: ID →→ street, city  (MVD)
```

교수가 여러 부서와 여러 주소를 가질 때, 부서 정보와 주소 정보는 독립적이므로 모든 조합의 튜플이 필요 → 중복 발생.

**MVD 정의**: `α →→ β` (α는 β를 다중값으로 결정)

r(R)에서 α →→ β가 성립 ↔ t₁[α] = t₂[α]이면 다음을 만족하는 t₃, t₄가 r에 존재:

```
t₃[α] = t₄[α] = t₁[α] = t₂[α]
t₃[β] = t₁[β],   t₃[R−α−β] = t₂[R−α−β]
t₄[β] = t₂[β],   t₄[R−α−β] = t₁[R−α−β]
```

직관: α와 β의 관계는 α와 R−α−β의 관계와 **독립적**.

```
α     β       R−α−β
━━━━━━━━━━━━━━━━━━━
a     b₁      c₁     ← t₁
a     b₂      c₂     ← t₂
a     b₁      c₂     ← t₃ (필요)
a     b₂      c₁     ← t₄ (필요)
```

**MVD 관련 규칙**:
- FD이면 MVD: α → β이면 α →→ β
- 보완 규칙: α →→ β이면 α →→ R − α − β

**자명한 MVD**: β ⊆ α이거나 β ∪ α = R이면 trivial

### 7.6.2 제4 정규형 (4NF, Fourth Normal Form)

**정의**: 릴레이션 스키마 R이 FD+MVD 집합 D에 대해 4NF이려면, D⁺의 모든 MVD `α →→ β`에 대해:

```
1) α →→ β가 trivial   OR
2) α가 R의 슈퍼키
```

**BCNF와의 관계**: 모든 4NF 스키마는 BCNF. BCNF이지만 4NF가 아닌 경우 존재.

```
증명: BCNF 위반 FD α→β가 있으면 α→β → α→→β이고 α가 슈퍼키 아님 → 4NF 위반
```

### 7.6.3 4NF 분해 알고리즘

BCNF 알고리즘과 구조 동일, FD 대신 MVD 사용:

```
result = {R}
done = false
while not done:
    if Ri가 4NF 아님:
        α →→ β를 4NF 위반 비자명 MVD로 선택 (α ∩ β = ∅)
        result = (result − {Ri}) ∪ {Ri − β} ∪ {α, β}
    else:
        done = true
```

**무손실 조건** (MVD 기반):

```
R1 ∩ R2 →→ R1   또는   R1 ∩ R2 →→ R2
```

**예**:
```
r2(ID, dept_name, street, city)
ID →→ dept_name 이 비자명 MVD, ID는 슈퍼키 아님

→ (ID, dept_name)
→ (ID, street, city)
→ 중복 해결, 4NF ✓
```

---

## 7.7 더 높은 정규형들 (More Normal Forms)

| 정규형 | 기반 제약 | 특징 |
|--------|----------|------|
| BCNF | 함수 종속성 | 모든 FD 기반 중복 제거 |
| 4NF | 다중값 종속성 | MVD 기반 중복 추가 제거 |
| PJNF/5NF | 조인 종속성 (Join Dependency) | MVD의 일반화 |
| DKNF | 도메인-키 제약 | 가장 일반적인 정규형 |

PJNF와 DKNF는 추론 규칙 시스템의 불완전성 문제로 실용적으로 거의 사용되지 않는다.

---

## 요약: 핵심 개념 정리

```
[정규화 의사결정 트리]

스키마 R, FD 집합 F
    ↓
BCNF 검사: 모든 비자명 α→β에서 α가 슈퍼키?
    ↓ YES                    ↓ NO
  BCNF ✓             의존성 보존 가능?
                         ↓ YES         ↓ NO
                       BCNF 분해    3NF 합성 알고리즘
                       (lossless)   (lossless + dep.preserving)
```

```
[핵심 수식]

무손실 분해: ΠR₁(r) ⋈ ΠR₂(r) = r
무손실 조건: R₁∩R₂ → R₁  또는  R₁∩R₂ → R₂

BCNF: 모든 비자명 α→β에 대해 α는 슈퍼키
3NF:  BCNF + β−α의 각 속성이 후보키에 포함 허용
4NF:  모든 비자명 α→→β에 대해 α는 슈퍼키

α⁺ 계산: F의 FD를 적용해 α로부터 도달 가능한 속성 집합

Armstrong 공리:
  반사율: β⊆α → α→β
  확장율: α→β → γα→γβ
  이행율: α→β, β→γ → α→γ

정준 커버: 불필요 속성 없음 + 유일한 좌변 + F⁺ = Fc⁺
```

```
[BCNF vs 3NF 선택 기준]

BCNF 선택: 의존성 보존이 가능할 때, 또는 SQL 레벨 검사가 가능할 때
3NF 선택: BCNF가 의존성 비보존을 유발할 때 (예: dept_advisor)
4NF: BCNF를 만족하나 MVD로 인한 중복이 있을 때
```
