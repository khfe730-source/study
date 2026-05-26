# Chapter 6: Database Design Using the E-R Model

## 개요

데이터베이스 스키마를 어떻게 설계할 것인가를 다루는 첫 번째 챕터. E-R(Entity-Relationship) 모델을 통해 엔티티와 관계를 식별하고, 최종적으로 관계형 스키마로 변환하는 전 과정을 다룬다.

---

## 6.1 설계 프로세스 개요 (Overview of the Design Process)

### 6.1.1 설계 단계 (Design Phases)

```
요구사항 수집
    ↓
개념적 설계 (Conceptual Design)
    → E-R 다이어그램 작성
    → 엔티티, 속성, 관계 명시
    ↓
논리적 설계 (Logical Design)
    → E-R → 관계형 스키마 변환
    → 구현 데이터 모델(관계형)로 매핑
    ↓
물리적 설계 (Physical Design)
    → 파일 조직, 인덱스 선택
    → Chapter 13, 14에서 상세 다룸
```

**핵심 포인트**: 논리적 스키마 변경은 응용 코드 전반에 영향을 주므로 물리적 설계보다 훨씬 비용이 크다. 초기 설계를 신중하게 해야 한다.

### 6.1.2 설계 대안과 함정 (Design Alternatives & Pitfalls)

나쁜 설계가 유발하는 두 가지 문제:

1. **중복성 (Redundancy)**: 동일 정보가 여러 곳에 저장 → 갱신 이상 (update anomaly)
   - 예: 섹션(section)마다 코스 제목을 반복 저장하면 일관성 위반 가능

2. **불완전성 (Incompleteness)**: 특정 상황을 모델링할 수 없음
   - 예: 코스 오퍼링만 존재하고 코스 자체 엔티티가 없으면, 개설되지 않은 신규 코스를 표현 불가능

---

## 6.2 E-R 모델 (The Entity-Relationship Model)

E-R 모델의 세 가지 기본 개념: **엔티티 집합(Entity Set)**, **관계 집합(Relationship Set)**, **속성(Attribute)**

### 6.2.1 엔티티 집합 (Entity Sets)

- **엔티티(Entity)**: 실세계에서 고유하게 식별 가능한 "사물" 또는 "객체"
- **엔티티 집합(Entity Set)**: 같은 타입의 엔티티들의 집합 (= 릴레이션의 스키마에 대응)
- **익스텐션(Extension)**: 엔티티 집합에 실제로 속하는 엔티티들의 모음 (= 릴레이션 인스턴스에 대응)

E-R 다이어그램에서 엔티티 집합은 **직사각형(Rectangle)**으로 표현:

```
┌──────────────┐    ┌──────────────┐
│  instructor  │    │   student    │
├──────────────┤    ├──────────────┤
│ ID           │    │ ID           │
│ name         │    │ name         │
│ salary       │    │ tot_cred     │
└──────────────┘    └──────────────┘
```
*(기본 키 속성은 밑줄로 표시)*

### 6.2.2 관계 집합 (Relationship Sets)

- **관계(Relationship)**: 여러 엔티티 간의 연관(association)
- **관계 집합(Relationship Set)**: 같은 타입의 관계들의 집합

공식 정의: 엔티티 집합 E₁, E₂, …, Eₙ에 대해 관계 집합 R은 다음의 부분 집합:

```
R ⊆ {(e₁, e₂, …, eₙ) | e₁ ∈ E₁, e₂ ∈ E₂, …, eₙ ∈ Eₙ}
```

E-R 다이어그램에서 관계 집합은 **다이아몬드(Diamond)**로 표현:

```
┌──────────────┐         ┌──────────────┐
│  instructor  │◇advisor◇│   student    │
└──────────────┘         └──────────────┘
```

**역할(Role)**: 동일 엔티티 집합이 관계에 여러 번 참여할 때 명시적으로 지정
- 예: `prereq` 관계에서 `course_id`와 `prereq_id`로 역할 구분

**서술 속성(Descriptive Attribute)**: 관계 자체에 붙는 속성
- 예: `takes` 관계의 `grade` 속성

**관계의 차수(Degree)**:
- Binary (2): 가장 일반적
- Ternary (3): 예) `proj_guide(instructor, student, project)`
- n-ary: 일반적으로 바이너리로 분해 가능하나 항상 그런 것은 아님

---

## 6.3 복합 속성 (Complex Attributes)

속성의 4가지 분류:

| 분류 | 설명 | E-R 표기 |
|------|------|---------|
| **단순(Simple)** | 더 이상 분할 불가 | 일반 속성명 |
| **복합(Composite)** | 하위 속성으로 분할 가능 | 계층적 나열 |
| **다중값(Multivalued)** | 하나의 엔티티에 여러 값 | `{phone_number}` |
| **유도(Derived)** | 다른 속성으로부터 계산됨 | `age()` |

복합 속성 예시:

```
instructor
├── ID
├── name (composite)
│   ├── first_name
│   ├── middle_initial
│   └── last_name
├── address (composite)
│   ├── street (composite)
│   │   ├── street_number
│   │   ├── street_name
│   │   └── apt_number
│   ├── city
│   ├── state
│   └── zip
├── { phone_number }    ← 다중값
├── date_of_birth
└── age()              ← 유도 속성
```

**null 값**: 속성이 해당 엔티티에 적용되지 않거나(not applicable), 알 수 없을 때(missing/unknown) 사용

---

## 6.4 매핑 카디날리티 (Mapping Cardinalities)

이진 관계 집합의 네 가지 카디날리티:

| 타입 | A → B | B → A |
|------|-------|-------|
| **일대일 (1:1)** | at most one | at most one |
| **일대다 (1:N)** | any number | at most one |
| **다대일 (M:1)** | at most one | any number |
| **다대다 (M:N)** | any number | any number |

**E-R 다이어그램 표기**:
- 방향 화살표(→): "one" 쪽
- 무방향 선(—): "many" 쪽

```
one-to-many (instructor advises many students):
instructor ←━━━ advisor ━━━━ student

many-to-many:
instructor ━━━━ advisor ━━━━ student
```

### 참여 제약 (Participation Constraints)

- **전체 참여(Total)**: 엔티티 집합의 모든 엔티티가 관계에 참여 → **이중 선(double line)**으로 표기
- **부분 참여(Partial)**: 일부만 참여 (기본값) → 단일 선

```
instructor ━━━━ advisor ══════ student
                (student: total participation, 모든 학생은 advisor가 있어야 함)
```

### l..h 카디날리티 표기

- `l`: 최소 카디날리티 (1이면 total participation)
- `h`: 최대 카디날리티 (`*`이면 제한 없음)

```
instructor  0..*   advisor   1..1   student
(instructor는 0명 이상 지도, student는 정확히 1명의 advisor)
```

---

## 6.5 기본 키 (Primary Key)

### 6.5.1 엔티티 집합의 기본 키

릴레이션 스키마와 동일하게 슈퍼키, 후보키, 기본키 개념 적용. 기본 키 속성은 E-R 다이어그램에서 **밑줄**로 표시.

### 6.5.2 관계 집합의 기본 키

관계 집합 R에 참여하는 엔티티 집합 E₁, …, Eₙ의 기본 키 합집합이 슈퍼키를 형성:

```
PK(E₁) ∪ PK(E₂) ∪ … ∪ PK(Eₙ)
```

카디날리티에 따른 최소 슈퍼키 선택:
- **M:N**: 양쪽 기본 키 합집합이 기본 키
- **1:N (A가 "many" 쪽)**: A의 기본 키가 기본 키
- **1:1**: 어느 쪽 기본 키든 선택 가능

### 6.5.3 약 엔티티 집합 (Weak Entity Sets)

독립적으로 기본 키를 가지지 못하고, **식별 엔티티 집합(Identifying Entity Set)**에 의존하는 엔티티:

- **판별자(Discriminator)**: 식별 엔티티의 기본 키와 결합하여 약 엔티티를 유일 식별하는 속성 집합 → **점선 밑줄**로 표시
- **식별 관계(Identifying Relationship)**: 약 엔티티를 강 엔티티와 연결하는 관계 → **이중 다이아몬드**로 표시
- 약 엔티티 집합의 기본 키 = 식별 엔티티 기본 키 + 판별자

```
┌═══════════════╗          ┌──────────────┐
║    section    ║          │    course    │
╠═══════════════╣          ├──────────────┤
║ sec_id        ║══════◇╗  │ course_id    │
║ semester      ║  sec_ ╠══│ title        │
║ year          ║ course║  │ credits      │
╚═══════════════╝        ╚═└──────────────┘
   (이중 사각형)          (이중 다이아몬드)
```

`section`의 기본 키 = `{course_id, sec_id, semester, year}`

식별 관계는:
- 약 엔티티 집합 → 식별 엔티티로 **many-to-one**
- 약 엔티티의 참여는 항상 **total**

---

## 6.6 엔티티 집합의 중복 속성 제거

E-R 모델에서 관계 집합으로 표현되는 연결은 엔티티 집합에서 해당 FK 속성을 제거해야 한다.

**예시**: `instructor`에 `dept_name` 속성을 두는 것이 아니라, `inst_dept` 관계 집합으로 표현:

```
Before (bad): instructor(ID, name, dept_name, salary)
After  (good): instructor(ID, name, salary)
               + inst_dept relationship
```

대학 데이터베이스의 최종 엔티티 집합 (중복 제거 후):

| 엔티티 집합 | 속성 | 기본 키 |
|------------|------|--------|
| classroom | building, room_number, capacity | (building, room_number) |
| department | dept_name, building, budget | dept_name |
| course | course_id, title, credits | course_id |
| instructor | ID, name, salary | ID |
| section | course_id, sec_id, semester, year | (course_id, sec_id, semester, year) |
| student | ID, name, tot_cred | ID |
| time_slot | time_slot_id, {(day, start_time, end_time)} | time_slot_id |

관계 집합 목록: `inst_dept`, `stud_dept`, `teaches`, `takes(grade)`, `course_dept`, `sec_course`, `sec_class`, `sec_time_slot`, `advisor`, `prereq`

---

## 6.7 E-R 다이어그램 → 관계형 스키마 변환

### 6.7.1 강 엔티티 집합 (Strong Entity Sets)

단순 속성만 있는 경우: 엔티티 집합명 = 스키마명, 속성 그대로 매핑

```
student entity set → student(ID, name, tot_cred)
```

### 6.7.2 복합/다중값 속성을 가진 강 엔티티 집합

- **복합 속성(Composite)**: 단순화(flatten) → 컴포넌트 속성들만 스키마에 포함 (복합 속성 자체는 미포함)
- **유도 속성(Derived)**: 스키마에 미포함 (stored procedure나 view로 처리)
- **다중값 속성(Multivalued)**: **별도 릴레이션 생성**

  ```
  instructor의 {phone_number} → instructor_phone(ID, phone_number)
  PK = (ID, phone_number), FK: ID → instructor
  ```

  특수 케이스: 엔티티 집합이 기본 키 속성 하나 + 다중값 속성 하나만 있는 경우, 기본 키 스키마는 생략 가능:
  ```
  time_slot entity set → time_slot(time_slot_id, day, start_time, end_time)
  (time_slot_id만의 별도 릴레이션 불필요)
  ```

### 6.7.3 약 엔티티 집합 (Weak Entity Sets)

약 엔티티 집합 A (속성 a₁,…,aₘ), 식별 엔티티 집합 B (기본 키 b₁,…,bₙ):

```
스키마 A: {a₁,…,aₘ} ∪ {b₁,…,bₙ}
기본 키: {b₁,…,bₙ} ∪ {discriminator attributes}
FK: b₁,…,bₙ → B
```

```sql
-- 예시
section(course_id, sec_id, semester, year)
  PK: (course_id, sec_id, semester, year)
  FK: course_id REFERENCES course(course_id)
```

### 6.7.4 관계 집합 (Relationship Sets)

관계 집합 R, 참여 엔티티의 기본 키 합집합 + 서술 속성:

```
advisor: instructor(PK=ID), student(PK=ID)
→ advisor(s_ID, i_ID)   [속성명 충돌 시 rename]
PK: s_ID (many-to-one from student to instructor이므로)
FK: s_ID → student, i_ID → instructor
```

자주 등장하는 스키마들:

```
teaches(ID, course_id, sec_id, semester, year)
takes(ID, course_id, sec_id, semester, year, grade)
prereq(course_id, prereq_id)   [role names 사용]
advisor(s_ID, i_ID)
```

### 6.7.5 스키마 중복성 (Redundancy)

**약 엔티티 집합의 식별 관계 스키마는 중복이므로 생략**:
- `sec_course`의 속성 = `section`의 기본 키 속성 → `section` 스키마와 중복 → 제거

### 6.7.6 스키마 합병 (Combination of Schemas)

**조건**: many-to-one 관계 + "many" 쪽의 전체 참여(total participation)

→ "many" 쪽 엔티티 스키마에 관계 스키마를 **합병**

```
inst_dept + instructor → instructor(ID, name, dept_name, salary)
stud_dept + student   → student(ID, name, dept_name, tot_cred)
course_dept + course  → course(course_id, title, dept_name, credits)
sec_class + section   → section(course_id, sec_id, semester, year, building, room_number)
sec_time_slot + section → section(course_id, sec_id, semester, year, building, room_number, time_slot_id)
```

부분 참여(partial)인 경우에도 합병 가능 (null 값 허용).

---

## 6.8 확장 E-R 기능 (Extended E-R Features)

### 6.8.1 특수화 (Specialization)

엔티티 집합 내의 서브그룹을 구분하는 **하향식(top-down)** 설계 프로세스.

- **ISA 관계**: "is a" 관계 → E-R 다이어그램에서 **속이 빈 화살표(hollow arrowhead)**로 표시
- 상위 엔티티 집합: **슈퍼클래스(superclass)**
- 하위 엔티티 집합: **서브클래스(subclass)**

```
       person
      /      \
 employee   student       ← ISA 화살표 (hollow)
  /    \
instr  secretary
```

**특수화 타입**:
- **Disjoint(비중복)**: 엔티티는 최대 하나의 서브클래스에만 속함 → 단일 화살표
- **Overlapping(중복)**: 여러 서브클래스 동시 소속 가능 → 별도 화살표

### 6.8.2 일반화 (Generalization)

여러 엔티티 집합의 공통 특성을 추출하여 상위 엔티티 집합을 합성하는 **상향식(bottom-up)** 과정. E-R 다이어그램에서 특수화와 동일하게 표현.

### 6.8.3 속성 상속 (Attribute Inheritance)

서브클래스는 슈퍼클래스의 모든 속성과 관계 집합 참여를 **상속(inherit)**받는다.

```
person: ID, name, street, city
  ↓ 상속
employee: ID, name, street, city + salary
  ↓ 상속
instructor: ID, name, street, city, salary + rank
```

- **단일 상속(Single Inheritance)**: 한 슈퍼클래스에서만 상속 → 계층(hierarchy)
- **다중 상속(Multiple Inheritance)**: 여러 슈퍼클래스에서 상속 → 격자(lattice)

### 6.8.4 특수화/일반화 제약 조건

1. **겹침 제약(Overlap Constraint)**:
   - Disjoint: 최대 하나의 서브클래스
   - Overlapping: 여러 서브클래스 가능

2. **완전성 제약(Completeness Constraint)**:
   - Total: 모든 슈퍼클래스 엔티티는 반드시 서브클래스에 속해야 함 (E-R에서 `total` 키워드 + 점선)
   - Partial(기본값): 서브클래스에 속하지 않는 슈퍼클래스 엔티티 허용

조합: partial-disjoint, partial-overlapping, total-disjoint, total-overlapping

### 6.8.5 집합체 (Aggregation)

**문제**: E-R 기본 모델은 관계 집합들 간의 관계를 직접 표현 불가.

**해결**: 관계 집합 자체를 상위 레벨 엔티티처럼 취급하여 다른 관계의 참여자로 사용.

```
                 project
                    |
instructor ◇ proj_guide ◇ student
                    |
              [aggregation]
                    |
                 eval_for
                    |
                evaluation
```

`proj_guide` 관계 집합 전체를 하나의 "엔티티"처럼 박스로 묶고, `eval_for` 관계로 `evaluation` 엔티티와 연결.

### 6.8.6 확장 기능의 관계형 스키마 변환

**일반화 → 릴레이션**:

**방법 1** (항상 적용 가능):
```sql
person(ID, name, street, city)
employee(ID, salary)       -- FK: ID → person
student(ID, tot_cred)      -- FK: ID → person
```

**방법 2** (disjoint + total인 경우에만):
```sql
employee(ID, name, street, city, salary)
student(ID, name, street, city, tot_cred)
-- person 릴레이션 불필요
-- 단, person을 참조하는 FK가 있으면 person 릴레이션 필요
```

방법 2의 단점: overlapping specialization에서 중복 저장 발생, FK 제약 정의 어려움.

**집합체 → 릴레이션**: 집합체 내부 관계 집합의 기본 키가 집합체의 기본 키가 됨. 집합체 자체에 대한 별도 스키마 불필요.

---

## 6.9 E-R 설계 이슈 (Design Issues)

### 6.9.1 흔한 실수 (Common Mistakes)

1. **FK를 속성으로 표현**: `student`에 `dept_name` 속성 직접 추가 → **잘못됨**
   - 올바른 방법: `stud_dept` 관계 집합 사용

2. **관계 집합에 참여 엔티티의 기본 키 속성 포함**: `advisor` 관계에 `student.ID`와 `instructor.ID`를 명시적으로 추가 → **잘못됨** (이미 암시적으로 포함됨)

3. **단일값 속성을 써야 할 곳에 다중값 사용 실수** (또는 그 반대):
   - `takes` 관계에 `assignment`, `marks` 속성 추가 → 학생-섹션 쌍당 과제 하나밖에 표현 불가
   - 올바른 방법: `assignment` 엔티티 집합 + `marks_in` 관계 사용

### 6.9.2 엔티티 집합 vs. 속성

어떤 개념을 **속성**으로 표현할지 **엔티티 집합**으로 표현할지:

- **속성으로**: 간단한 값, 구조 없음
- **엔티티 집합으로**: 다른 엔티티와의 관계가 필요하거나, 자체 속성을 가질 때

예: `phone_number`가 단순 연락처이면 속성. 전화기 소유자, 위치 등 추가 정보가 필요하면 `phone` 엔티티.

### 6.9.3 엔티티 집합 vs. 관계 집합

어떤 개념을 **엔티티 집합**으로 표현할지 **관계 집합**으로 표현할지:

- 일반적 지침: "동사"는 관계, "명사"는 엔티티
- 그러나 고객-상품 간의 "판매(sale)"처럼 자체 속성을 가지고 여러 엔티티와 연결되어야 하면 엔티티로 모델링

### 6.9.4 이진 vs. n진 관계 집합

n진 관계 집합 → 이진 관계로 분해 가능한 경우가 많음. 그러나 항상 가능하지는 않으며, 의미가 달라질 수 있음.

---

## 6.10 다른 표기법 (Alternative Notations)

### UML 클래스 다이어그램 비교

| E-R | UML |
|-----|-----|
| 엔티티 집합 | 클래스 |
| 관계 집합 | 연관(association) |
| 다중값 속성 | 별도 클래스 |
| ISA (hollow arrow) | 일반화 화살표 |
| 집합체 | 없음 (별도 모델링) |

### Crow's-Foot 표기법

```
|━━━━◁── one (exactly one)
o━━━━◁── zero or one
|━━━━━<  one-to-many
o━━━━━<  zero-to-many
```

### Chen 표기법

원래 E-R 표기법. 카디날리티를 관계 선 위에 1, M, N으로 표기.

---

## 요약 & 핵심 패턴

```
[E-R 설계 결정 트리]

개념 표현 방법?
├── 단순 값 → 속성
├── 고유 식별 + 다른 것과 관계 → 엔티티
│   ├── 독립적 기본 키 있음 → 강 엔티티
│   └── 식별 엔티티 필요 → 약 엔티티 (이중 사각형)
└── 엔티티 간 연결 → 관계
    ├── 자체 속성 있음 → 서술 속성 추가
    └── 관계 자체를 다른 관계에 참여시켜야 함 → 집합체

[E-R → 스키마 변환 요약]

강 엔티티        → 동명 스키마 (단순 속성 그대로)
복합 속성        → 평탄화 (flatten)
다중값 속성      → 별도 스키마 생성
약 엔티티        → 식별 엔티티 PK + discriminator
관계 집합        → 참여 엔티티 PK 합집합 + 서술 속성
식별 관계 스키마 → 중복이므로 제거
M:1 + total     → "many" 엔티티 스키마에 합병
일반화 (항상)    → 슈퍼클래스 + 서브클래스 각각 (FK로 참조)
일반화 (disjoint+total) → 서브클래스에 슈퍼클래스 속성 병합 가능
집합체           → 정의 관계 집합의 PK = 집합체 PK
```
