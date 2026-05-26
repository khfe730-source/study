# Chapter 2. Introduction to the Relational Model

## 개요

관계형 모델(Relational Model)은 1970년 E. F. Codd가 제안한 이후 반세기 넘게 상업적 데이터 처리의 지배적 모델로 자리잡고 있다. 네트워크 모델·계층 모델보다 단순하며, 오브젝트-관계형 기능(복합 타입, 저장 프로시저), XML/반구조적 데이터 지원, 컬럼 스토어까지 흡수하며 생명력을 이어가고 있다. 이 챕터에서는 관계형 모델의 수학적 기초와 관계 대수(Relational Algebra)를 다룬다.

---

## 2.1 관계형 데이터베이스의 구조 (Structure of Relational Databases)

### 핵심 용어

| 수학적 용어 | 관계형 모델 용어 | 설명 |
|------------|----------------|------|
| 관계(Relation) | 테이블(Table) | 튜플들의 집합 |
| 튜플(Tuple) | 행(Row) | 관련 값들의 순서열 |
| 속성(Attribute) | 열(Column) | 관계의 각 항목 |
| 관계 인스턴스(Relation Instance) | 테이블의 현재 내용 | 특정 시점의 튜플 집합 |
| 도메인(Domain) | 속성의 허용 값 집합 | 각 속성에 허용되는 모든 가능한 값의 집합 |

### 도메인 원자성 (Atomic Domain)

도메인이 **원자적(atomic)**이려면, 도메인의 각 원소가 더 이상 쪼갤 수 없는 단위여야 한다.

- **위반 예시**: `phone_number` 속성이 전화번호 집합을 저장하는 경우 → 원자적이지 않음
- **허용 예시**: 전화번호 하나만 저장하되 단일 단위로 취급하는 경우 → 원자적

> 관계형 모델의 **1NF(제1정규형)**은 모든 속성의 도메인이 원자적임을 요구한다.

### Null 값

`null`은 값이 알 수 없거나 존재하지 않음을 나타내는 특수 값이다. null은 각종 연산에서 복잡한 문제를 야기하므로 가능하면 사용을 피해야 한다 (Section 3.6에서 상세 논의).

### 대학교 예제 스키마

본 챕터 전반에 걸쳐 사용하는 대학교 데이터베이스의 전체 릴레이션:

```
classroom  (building, room_number, capacity)
department (dept_name, building, budget)
course     (course_id, title, dept_name, credits)
instructor (ID, name, dept_name, salary)
section    (course_id, sec_id, semester, year, building, room_number, time_slot_id)
teaches    (ID, course_id, sec_id, semester, year)
student    (ID, name, dept_name, tot_cred)
takes      (ID, course_id, sec_id, semester, year, grade)
advisor    (s_ID, i_ID)
time_slot  (time_slot_id, day, start_time, end_time)
prereq     (course_id, prereq_id)
```

---

## 2.2 데이터베이스 스키마 (Database Schema)

- **스키마(Schema)**: 릴레이션의 논리적 설계. 속성 목록과 각 속성의 도메인으로 구성. 프로그래밍 언어의 **타입 정의**에 해당
- **인스턴스(Instance)**: 특정 시점의 실제 데이터. 프로그래밍 언어의 **변수 값**에 해당

```
relation schema  ↔  type definition
relation instance ↔  variable value
```

스키마는 거의 변하지 않지만, 인스턴스는 데이터가 추가·삭제·수정될 때마다 바뀐다.

공통 속성을 서로 다른 릴레이션 스키마에 포함시키는 것이 릴레이션 간 연결 수단이 된다. 예를 들어 `dept_name`은 `instructor`와 `department` 양쪽에 존재하여 두 릴레이션을 연결하는 다리 역할을 한다.

---

## 2.3 키 (Keys)

### 수퍼키 (Superkey)

릴레이션 r에서 **튜플을 유일하게 식별**할 수 있는 하나 이상의 속성 집합.

- `instructor`에서 `{ID}`, `{ID, name}`, `{ID, name, dept_name}` 등은 모두 수퍼키
- 수퍼키의 모든 상위집합(superset)도 수퍼키

### 후보키 (Candidate Key)

**진부분집합(proper subset) 중 수퍼키가 없는** 최소 수퍼키.

- `instructor`에서 `{ID}`와 `{name, dept_name}` (이름+학과가 유일하다면) 모두 후보키
- `{ID, name}`은 후보키가 아님 — `{ID}` 자체가 수퍼키이므로

### 기본키 (Primary Key)

데이터베이스 설계자가 **튜플을 식별하는 주요 수단으로 선택한** 후보키.

- 스키마 표기 시 기본키 속성에 밑줄: `instructor(ID, name, dept_name, salary)`
- 기본키 속성은 관례상 다른 속성보다 먼저 나열
- **기본키는 관계 전체의 제약 조건**이지 개별 튜플의 속성이 아님
- 기본키 값은 자주 변경되지 않아야 함 (예: 주소는 부적합, 고유 생성 ID는 적합)

```
복합 기본키 예시:
  classroom(building, room_number, capacity)
  → 기본키: {building, room_number}  — 둘 중 하나만으로는 유일 식별 불가

  time_slot(time_slot_id, day, start_time, end_time)
  → 기본키: {time_slot_id, day, start_time}  — 하나의 time_slot_id가 요일별로 여러 시간대 가능
```

### 외래키 (Foreign Key) & 참조 무결성 (Referential Integrity)

**외래키 제약(Foreign-Key Constraint)**: 릴레이션 r₁의 속성 집합 A의 값이, 릴레이션 r₂의 기본키 B에 반드시 존재해야 한다.

```
instructor.dept_name  →(FK)→  department.dept_name (PK)
section.{building, room_number}  →(FK)→  classroom.{building, room_number} (PK)
```

**참조 무결성 제약(Referential Integrity Constraint)**: 외래키보다 더 일반적인 제약. 참조되는 속성이 기본키가 아니어도 됨.

```
section.time_slot_id  →(참조 무결성)→  time_slot.time_slot_id
  ※ time_slot_id는 time_slot의 기본키가 아니라 기본키의 일부이므로
    외래키 제약이 아닌 참조 무결성 제약으로만 표현 가능
```

> 현재 대부분의 DBMS는 외래키 제약은 지원하지만, **기본키가 아닌 속성에 대한 참조 무결성 제약은 지원하지 않는** 경우가 많다.

---

## 2.4 스키마 다이어그램 (Schema Diagrams)

스키마와 키 제약을 시각화하는 다이어그램.

표기 규칙:
- 각 릴레이션은 박스로 표현
- 기본키 속성에 **밑줄**
- 외래키 제약: **단방향 화살표** (참조하는 릴레이션 → 참조되는 릴레이션의 기본키)
- 참조 무결성 제약(기본키가 아닌 경우): **양방향 화살표**

```
대학교 데이터베이스 스키마 다이어그램 (ASCII 표현)

 ┌─────────────────┐         ┌──────────────────┐
 │   instructor    │         │    department    │
 ├─────────────────┤         ├──────────────────┤
 │ ID (PK)         │         │ dept_name (PK)   │
 │ name            │         │ building         │
 │ dept_name ──────┼────────▶│ budget           │
 │ salary          │         └──────────────────┘
 └────────┬────────┘
          │                   ┌──────────────────┐
          │                   │     teaches      │
          │                   ├──────────────────┤
          └──────────────────▶│ ID               │
                              │ course_id ───┐   │
                              │ sec_id       │   │
                              │ semester     │   │
                              │ year         │   │
                              └──────────────┼───┘
                                             │
                              ┌──────────────▼───┐
                              │     section      │
                              ├──────────────────┤
                              │ course_id (PK)   │
                              │ sec_id (PK)      │
                              │ semester (PK)    │
                              │ year (PK)        │
                              │ building ────┐   │
                              │ room_number ─┼───┼──▶ classroom
                              │ time_slot_id─┼───┼──▷ time_slot (참조무결성)
                              └─────────────────┘
```

---

## 2.5 관계형 질의 언어 (Relational Query Languages)

질의 언어의 세 분류:

| 분류 | 설명 | 예시 |
|------|------|------|
| **명령형(Imperative)** | 어떤 순서로 연산을 수행할지 명시. 상태 변수를 갱신 | - |
| **함수형(Functional)** | 함수 평가로 계산을 표현. 부작용(side-effect) 없음 | 관계 대수 |
| **선언형(Declarative)** | 원하는 정보만 기술, 획득 방법은 DBMS가 결정 | 튜플 관계 칼큘러스, 도메인 관계 칼큘러스 |

실용 질의 언어(SQL)는 세 방식을 혼합한다. 관계 대수는 SQL의 이론적 기반이다.

---

## 2.6 관계 대수 (Relational Algebra)

관계 대수는 하나 또는 두 개의 릴레이션을 입력받아 새로운 릴레이션을 반환하는 **연산들의 집합**이다. 연산 결과가 다시 릴레이션이므로 연산들을 자유롭게 합성(compose)할 수 있다.

> 관계 대수는 집합(set) 의미론을 따른다: 릴레이션은 중복 없는 튜플의 집합이며, 연산 결과에서 중복은 제거된다.

### 2.6.1 셀렉트 (Select) — σ

**정의**: 주어진 조건(predicate)을 만족하는 튜플만 선택.

```
σ₍조건₎(릴레이션)
```

**예시**:
```
σ_dept_name="Physics"(instructor)
  → Physics 학과 교수만 반환

σ_salary>90000(instructor)
  → 급여 $90,000 초과 교수

σ_dept_name="Physics" ∧ salary>90000(instructor)
  → Physics 학과이면서 급여 $90,000 초과

σ_dept_name=building(department)
  → 학과명과 건물명이 같은 학과
```

조건에 사용 가능한 비교 연산자: `=, ≠, <, ≤, >, ≥`  
논리 연산자: `∧(and), ∨(or), ¬(not)`

### 2.6.2 프로젝트 (Project) — Π

**정의**: 지정한 속성들만 남기고 나머지를 제거. 결과에서 중복 튜플은 제거됨(집합 의미론).

```
Π₍속성목록₎(릴레이션)
```

**예시**:
```
Π_ID,name,salary(instructor)
  → ID, 이름, 급여만 포함한 릴레이션

Π_ID,name,salary/12(instructor)   ← 일반화 프로젝트
  → 월급(salary÷12) 계산 포함
```

### 2.6.3 연산의 합성 (Composition)

관계 대수 연산의 결과가 릴레이션이므로 연산들을 중첩할 수 있다. 산술 표현식처럼 자유롭게 조합 가능.

```
Π_name(σ_dept_name="Physics"(instructor))
  → Physics 학과 교수들의 이름
```

### 2.6.4 카르테시안 곱 (Cartesian Product) — ×

**정의**: 두 릴레이션의 모든 튜플 쌍을 연결. r₁이 n₁개, r₂가 n₂개의 튜플을 가지면 결과는 n₁×n₂개.

```
instructor × teaches
```

동일한 속성명이 두 릴레이션에 존재하면 `릴레이션명.속성명`으로 구별:
```
(instructor.ID, name, dept_name, salary, teaches.ID, course_id, sec_id, semester, year)
```

카르테시안 곱 자체는 모든 조합을 생성하므로 실제 의미 있는 결과를 얻으려면 이후 셀렉트가 필요하다.

### 2.6.5 조인 (Join) — ⋈

**정의**: 카르테시안 곱 + 셀렉트를 하나의 연산으로 결합.

```
r ⋈_θ s  =  σ_θ(r × s)
```

**예시**: 교수가 실제로 가르친 과목 정보를 결합
```
instructor ⋈_instructor.ID=teaches.ID teaches
  =  σ_instructor.ID=teaches.ID(instructor × teaches)
```

카르테시안 곱이 12×15=180개 튜플을 생성하는 반면, 조인은 실제 교수-수업 연결 튜플(15개)만 반환한다. 과목을 가르치지 않은 교수(Gold, Caliﬁeri, Singh)는 결과에서 제외됨.

### 2.6.6 집합 연산 (Set Operations)

집합 연산의 전제 조건: 두 입력 릴레이션이 **호환(compatible)** 해야 함.
1. 속성 수(arity)가 동일
2. 대응하는 속성의 타입이 동일

| 연산 | 기호 | 설명 |
|------|------|------|
| **합집합(Union)** | ∪ | 두 릴레이션 중 어느 쪽에든 속한 튜플 |
| **교집합(Intersection)** | ∩ | 두 릴레이션 모두에 속한 튜플 |
| **차집합(Set Difference)** | − | 첫 번째에는 있고 두 번째에는 없는 튜플 |

**예시**:
```
-- Fall 2017 또는 Spring 2018에 개설된 과목
Π_course_id(σ_semester="Fall" ∧ year=2017(section))
∪ Π_course_id(σ_semester="Spring" ∧ year=2018(section))

-- 두 학기 모두 개설된 과목
Π_course_id(σ_semester="Fall" ∧ year=2017(section))
∩ Π_course_id(σ_semester="Spring" ∧ year=2018(section))

-- Fall 2017에만 개설된 과목 (Spring 2018 제외)
Π_course_id(σ_semester="Fall" ∧ year=2017(section))
− Π_course_id(σ_semester="Spring" ∧ year=2018(section))
```

### 2.6.7 배정 연산 (Assignment) — ←

복잡한 관계 대수 식을 임시 릴레이션 변수에 쪼개어 작성하기 위한 편의 연산. 추가적인 표현력을 제공하지는 않는다.

```
courses_fall_2017 ← Π_course_id(σ_semester="Fall" ∧ year=2017(section))
courses_spring_2018 ← Π_course_id(σ_semester="Spring" ∧ year=2018(section))
result ← courses_fall_2017 ∩ courses_spring_2018
```

### 2.6.8 리네임 연산 (Rename) — ρ

**정의**: 관계 대수 식의 결과에 이름을 부여하거나 속성명을 변경.

```
ρ_x(E)            → 식 E의 결과를 x라는 이름으로 반환
ρ_x(A1,...,An)(E) → 결과 이름을 x로, 속성명을 A1,...,An으로 변경
```

**사용 예**: 자기 자신과의 조인 (self-join). 동일 릴레이션을 두 번 참조할 때 구별 필요.

```
-- ID 12121(Wu)보다 급여가 높은 교수의 ID와 이름 찾기
Π_i.ID, i.name(
  σ_i.salary > w.salary(
    ρ_i(instructor) × σ_w.ID=12121(ρ_w(instructor))
  )
)
```

### 2.6.9 동등 질의 (Equivalent Queries)

동일한 결과를 내는 관계 대수 식이 여러 개 존재할 수 있다.

```
-- 방법 1: 조인 후 셀렉트
σ_dept_name="Physics"(instructor ⋈_instructor.ID=teaches.ID teaches)

-- 방법 2: 셀렉트 후 조인  ← 쿼리 최적화 시 더 효율적
(σ_dept_name="Physics"(instructor)) ⋈_instructor.ID=teaches.ID teaches
```

두 식은 결과가 동일하지만 성능이 다를 수 있다. **쿼리 옵티마이저(Query Optimizer)**는 이처럼 대수적으로 동등한 식들 중 가장 효율적인 실행 계획을 찾는다 (Chapter 16에서 상세 논의).

### 연산 요약표

| 연산 | 기호 | 단항/이항 | 설명 |
|------|------|:---------:|------|
| 셀렉트 | σ | 단항 | 조건을 만족하는 튜플 선택 |
| 프로젝트 | Π | 단항 | 지정 속성만 유지 |
| 리네임 | ρ | 단항 | 결과/속성에 이름 부여 |
| 카르테시안 곱 | × | 이항 | 두 릴레이션의 모든 튜플 쌍 결합 |
| 조인 | ⋈ | 이항 | 조건을 만족하는 튜플 쌍 결합 (= σ+×) |
| 합집합 | ∪ | 이항 | 두 릴레이션의 합집합 |
| 교집합 | ∩ | 이항 | 두 릴레이션의 교집합 |
| 차집합 | − | 이항 | 첫 번째에서 두 번째를 뺀 차집합 |
| 배정 | ← | - | 임시 변수에 식 할당 |

> **Note**: 이 외에 자연 조인(Natural Join ⋈), 외부 조인(Outer Join), 집계 연산(Aggregation)도 실용적으로 중요한 연산이다. 각각 Section 4.1.1, Section 4.1.3, Section 3.7에서 다룬다.

---

## 2.7 요약

- 관계형 모델의 핵심: 테이블(=릴레이션)의 집합. 각 릴레이션은 스키마(구조)와 인스턴스(데이터)로 구성
- **수퍼키 ⊇ 후보키 ⊇ 기본키** 관계로 유일 식별 속성을 계층화
- 외래키는 릴레이션 간 참조 무결성을 보장하는 핵심 메커니즘
- 스키마 다이어그램은 스키마와 제약을 시각적으로 표현
- 관계 대수는 SQL의 이론적 토대이며, 연산의 합성을 통해 복잡한 질의를 표현
- 쿼리 옵티마이저는 관계 대수의 대수적 동등성을 이용해 최적 실행 계획을 탐색

---

## 핵심 용어 정리

| 용어 | 설명 |
|------|------|
| Relation (릴레이션) | 테이블. 튜플의 집합 |
| Tuple (튜플) | 행. n개 값의 순서열 |
| Attribute (속성) | 열. 릴레이션의 각 항목 |
| Domain (도메인) | 속성의 허용 값 집합 |
| Atomic Domain | 원소가 더 이상 쪼개질 수 없는 도메인 (1NF 요건) |
| Schema | 릴레이션의 논리적 설계 (속성 목록 + 도메인) |
| Instance | 특정 시점의 실제 데이터 |
| Superkey | 튜플을 유일 식별하는 속성 집합 |
| Candidate Key | 최소 수퍼키 |
| Primary Key | 설계자가 선택한 후보키 |
| Foreign Key | 다른 릴레이션의 기본키를 참조하는 속성 |
| Referential Integrity | 참조되는 속성이 반드시 참조 릴레이션에 존재해야 함 |
| Schema Diagram | 스키마와 키 제약의 시각적 표현 |
| Relational Algebra | 릴레이션에 대한 대수 연산 집합 (SQL의 이론적 기반) |
| Select σ | 조건 만족 튜플 필터링 |
| Project Π | 속성 서브셋 추출 |
| Cartesian Product × | 두 릴레이션의 모든 튜플 조합 |
| Join ⋈ | 조건 만족 카르테시안 곱 |
| Union ∪ | 두 릴레이션의 합집합 |
| Set Difference − | 두 릴레이션의 차집합 |
| Intersection ∩ | 두 릴레이션의 교집합 |
| Assignment ← | 임시 릴레이션 변수 할당 |
| Rename ρ | 릴레이션/속성 이름 변경 |
| Query Optimizer | 동등 질의 중 최소 비용 실행 계획 선택 컴포넌트 |
