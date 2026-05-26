# Chapter 1. Introduction

## 개요

DBMS(Database Management System)는 상호 연관된 데이터의 집합과 그 데이터에 접근하는 프로그램들의 집합이다. 데이터베이스 시스템의 핵심 목표는 대용량 정보를 **편리하고 효율적으로** 저장·검색할 수 있는 환경을 제공하는 것이다. 복잡성 관리의 핵심은 **추상화(abstraction)**이며, DBMS는 저장 구조의 세부사항을 숨기고 사용자에게 단순화된 뷰를 제공한다.

---

## 1.1 데이터베이스 시스템 응용 분야 (Database-System Applications)

데이터베이스 시스템이 관리하는 데이터의 공통 특성:
- 높은 가치(highly valuable)
- 상대적으로 대용량(relatively large)
- 다수 사용자·애플리케이션의 동시 접근

### 주요 응용 분야
- 기업 정보 시스템: 판매, 회계, 인사
- 제조: 공급망(supply chain), 재고, 주문
- 금융/뱅킹: 계좌, 대출, 신용카드, 주식 거래
- 통신: 통화 기록, 청구서
- 웹 서비스: 소셜미디어, 온라인 쇼핑, 온라인 광고
- 내비게이션, 문서 데이터베이스 등

### 데이터베이스 사용 두 가지 모드
1. **온라인 트랜잭션 처리(OLTP, Online Transaction Processing)**: 다수의 사용자가 소량의 데이터를 읽고 쓰는 패턴
2. **데이터 분석(Data Analytics)**: 데이터에서 규칙과 패턴을 자동으로 발견하여 의사결정에 활용. 데이터 마이닝(data mining)과 결합된다.

---

## 1.2 데이터베이스 시스템의 목적 (Purpose of Database Systems)

파일 처리 시스템(file-processing system)의 한계가 DBMS 탄생의 근본 원인이다.

### 파일 처리 방식의 문제점

| 문제 | 설명 |
|------|------|
| **데이터 중복·불일치** (Data Redundancy & Inconsistency) | 여러 파일에 동일 정보가 중복 저장되어 일관성이 깨질 수 있음 |
| **데이터 접근의 어려움** (Difficulty in Accessing Data) | 새로운 질의가 필요할 때마다 프로그램을 별도 작성해야 함 |
| **데이터 고립** (Data Isolation) | 다양한 파일·포맷에 분산되어 통합 접근이 어려움 |
| **무결성 문제** (Integrity Problems) | 제약 조건(constraint)이 코드에 하드코딩되어 변경 시 여러 프로그램 수정 필요 |
| **원자성 문제** (Atomicity Problems) | 장애 발생 시 부분 실행 상태가 되어 데이터 불일치 발생 |
| **동시 접근 이상** (Concurrent-Access Anomalies) | 다수 사용자 동시 갱신 시 경쟁 조건(race condition)으로 데이터 오염 |
| **보안 문제** (Security Problems) | 접근 제어를 체계적으로 강제하기 어려움 |

> **예시 (원자성)**: 계좌 A→B로 $500 이체 중 장애 발생 시, A에서는 차감됐으나 B에는 입금 안 된 상태가 될 수 있다.
>
> **예시 (동시 접근)**: 두 창구 직원이 동시에 잔액 $10,000을 읽고 각각 $500, $100을 인출 처리하면 결과가 $9,400이 아닌 $9,500 또는 $9,900이 될 수 있다.

---

## 1.3 데이터의 뷰 (View of Data)

### 1.3.1 데이터 모델 (Data Models)

데이터 모델은 데이터, 데이터 관계, 의미론(semantics), 일관성 제약을 기술하는 개념적 도구의 집합이다.

| 모델 | 설명 |
|------|------|
| **관계형 모델** (Relational Model) | 테이블(relation)의 집합으로 데이터와 관계를 표현. 가장 광범위하게 사용 |
| **개체-관계 모델** (E-R Model) | 개체(entity)와 관계(relationship)로 표현. 데이터베이스 설계에 주로 사용 |
| **반구조적 데이터 모델** (Semi-structured) | 동일 타입의 데이터 항목이 서로 다른 속성을 가질 수 있음. JSON, XML |
| **객체 기반 모델** (Object-Based) | OOP 개념(캡슐화, 메서드, 객체 식별자)을 관계형 모델에 통합 |

### 1.3.2 관계형 데이터 모델 (Relational Data Model)

데이터를 테이블 형태로 표현한다. 각 행(row/tuple)은 하나의 레코드, 각 열(column/attribute)은 속성을 나타낸다.

```
instructor 테이블
+-------+----------+-----------+--------+
| ID    | name     | dept_name | salary |
+-------+----------+-----------+--------+
| 22222 | Einstein | Physics   | 95000  |
| 33456 | Gold     | Physics   | 87000  |
+-------+----------+-----------+--------+

department 테이블
+-----------+----------+--------+
| dept_name | building | budget |
+-----------+----------+--------+
| Biology   | Watson   | 90000  |
| Physics   | Watson   | 70000  |
+-----------+----------+--------+
```

### 1.3.3 데이터 추상화 (Data Abstraction)

3단계 추상화 계층:

```
┌─────────────────────────────────┐
│         View Level              │  ← 사용자별 맞춤 뷰 (subschema)
├─────────────────────────────────┤
│        Logical Level            │  ← 데이터 구조와 관계 (logical schema)
├─────────────────────────────────┤
│        Physical Level           │  ← 실제 저장 구조 (physical schema)
└─────────────────────────────────┘
```

- **물리적 수준(Physical level)**: 데이터가 실제로 어떻게 저장되는지 기술 (B-tree, 해시 등 저수준 구조)
- **논리적 수준(Logical level)**: 어떤 데이터가 저장되고 관계가 어떠한지 기술. 프로그래머가 주로 다루는 수준
- **뷰 수준(View level)**: 특정 사용자에게 필요한 데이터의 일부만 노출

**물리적 데이터 독립성(Physical Data Independence)**: 물리적 스키마가 변경되어도 논리적 스키마와 애플리케이션 프로그램을 수정하지 않아도 되는 성질.

### 1.3.4 인스턴스와 스키마 (Instances and Schemas)

- **스키마(Schema)**: 데이터베이스의 전체적인 설계/구조. 프로그래밍 언어의 타입 선언에 해당
- **인스턴스(Instance)**: 특정 시점에 데이터베이스에 실제로 저장된 데이터의 집합. 변수의 현재 값에 해당

---

## 1.4 데이터베이스 언어 (Database Languages)

DBMS는 **DDL(Data-Definition Language)**과 **DML(Data-Manipulation Language)**을 제공한다. 실제로는 SQL처럼 두 기능을 통합한 단일 언어로 구현된다.

### 1.4.1 데이터 정의 언어 (DDL)

스키마 정의 및 추가 속성(제약 조건 등) 명세에 사용한다.

DDL로 지정 가능한 무결성 제약(Integrity Constraints):

| 제약 | 설명 |
|------|------|
| **도메인 제약(Domain Constraints)** | 속성에 허용되는 값의 범위 지정 (타입, 범위 등) |
| **참조 무결성(Referential Integrity)** | 한 릴레이션의 값이 다른 릴레이션에 반드시 존재해야 함 (외래 키) |
| **권한(Authorization)** | 읽기/삽입/갱신/삭제 권한을 사용자별로 부여 |

DDL 처리 결과는 **데이터 딕셔너리(data dictionary)**에 저장된다. 데이터 딕셔너리는 메타데이터(data about data)를 보관하며, DBMS만이 직접 접근·수정할 수 있다.

```sql
-- SQL DDL 예시
CREATE TABLE department (
    dept_name  CHAR(20),
    building   CHAR(15),
    budget     NUMERIC(12,2)
);
```

### 1.4.2 데이터 조작 언어 (DML)

데이터 조작의 네 가지 유형: **조회(Retrieval)**, **삽입(Insertion)**, **삭제(Deletion)**, **수정(Modification)**

| 종류 | 설명 |
|------|------|
| **절차적 DML (Procedural)** | 어떤 데이터가 필요하고 **어떻게** 얻을지를 명시 |
| **선언적 DML (Declarative/Nonprocedural)** | 어떤 데이터가 필요한지만 명시, 획득 방법은 DBMS가 결정 |

선언적 DML이 배우기 쉽고 사용하기 편하지만, DBMS가 효율적인 접근 방법을 스스로 찾아야 한다. SQL은 비절차적(nonprocedural) 언어다.

```sql
-- SQL DML 예시: History 학과 교수 이름 조회
SELECT instructor.name
FROM instructor
WHERE instructor.dept_name = 'History';

-- 예산 $95,000 초과 학과 교수 ID 조회 (조인)
SELECT instructor.ID, department.dept_name
FROM instructor, department
WHERE instructor.dept_name = department.dept_name
  AND department.budget > 95000;
```

### 1.4.3 애플리케이션 프로그램에서의 데이터베이스 접근

SQL 자체는 튜링 완전(Turing-complete)하지 않으므로, C/C++, Java, Python 같은 호스트 언어(host language)에 SQL을 내장하여 사용한다.

- **ODBC(Open Database Connectivity)**: C 등의 언어를 위한 표준 API
- **JDBC(Java Database Connectivity)**: Java를 위한 표준 API

---

## 1.5 데이터베이스 설계 (Database Design)

데이터베이스 설계의 주요 단계:

```
요구사항 분석
    ↓
개념적 설계 (Conceptual Design)
│  ∙ E-R 모델 또는 정규화(Normalization) 활용
│  ∙ 어떤 데이터를 저장할지, 관계는 어떤지 정의
    ↓
논리적 설계 (Logical Design)
│  ∙ 개념적 스키마 → 구현 데이터 모델(예: 관계형)로 변환
    ↓
물리적 설계 (Physical Design)
   ∙ 파일 조직, 인덱스 구조 등 물리적 특성 결정
```

- **개념적 설계**: E-R 모델로 엔티티와 관계를 모델링, 중복 제거
- **기능적 요구사항 명세**: 데이터에 수행할 트랜잭션(조회, 수정, 삭제 등)을 기술
- 좋은 스키마 설계와 나쁜 스키마 설계를 구분하는 방법은 Chapter 7(정규화)에서 다룬다.

---

## 1.6 데이터베이스 엔진 (Database Engine)

DBMS의 기능 컴포넌트는 크게 세 부분으로 나뉜다:

```
┌──────────────────────────────────────────────────────┐
│                  Query Processor                     │
│  DDL Interpreter │ DML Compiler │ Query Eval Engine  │
├──────────────────────────────────────────────────────┤
│                 Storage Manager                      │
│  Auth/Integrity │ Transaction │ File │ Buffer Mgr    │
├──────────────────────────────────────────────────────┤
│              Transaction Manager                     │
│       Concurrency Control │ Recovery Manager         │
└──────────────────────────────────────────────────────┘
              ↕ Disk Storage
  [Data Files] [Data Dictionary] [Indices]
```

### 1.6.1 저장 관리자 (Storage Manager)

저수준 파일 시스템과 상위 애플리케이션/쿼리 사이의 인터페이스를 담당.

| 컴포넌트 | 역할 |
|----------|------|
| 권한·무결성 관리자 (Authorization & Integrity Manager) | 무결성 제약 검사, 접근 권한 확인 |
| 트랜잭션 관리자 (Transaction Manager) | 시스템 장애 시 일관성 보장, 동시 트랜잭션 충돌 방지 |
| 파일 관리자 (File Manager) | 디스크 공간 할당, 데이터 구조 관리 |
| 버퍼 관리자 (Buffer Manager) | 디스크↔메인 메모리 데이터 이동, 캐시 정책 결정. **크기가 메인 메모리를 초과하는 DB도 처리 가능하게 함** |

저장 관리자가 구현하는 자료 구조:
- **데이터 파일(Data files)**: DB 데이터 자체
- **데이터 딕셔너리(Data dictionary)**: 스키마 등 메타데이터
- **인덱스(Indices)**: 빠른 데이터 접근을 위한 포인터 구조

### 1.6.2 쿼리 프로세서 (Query Processor)

| 컴포넌트 | 역할 |
|----------|------|
| DDL 해석기 (DDL Interpreter) | DDL 문 해석 후 데이터 딕셔너리에 기록 |
| DML 컴파일러 (DML Compiler) | DML 쿼리를 저수준 실행 계획으로 변환. **쿼리 최적화(Query Optimization)** 수행 |
| 쿼리 평가 엔진 (Query Evaluation Engine) | 실행 계획을 실제로 수행 |

쿼리 최적화(Query Optimization): 동일 결과를 내는 여러 실행 계획 중 최소 비용을 선택.

### 1.6.3 트랜잭션 관리 (Transaction Management)

트랜잭션(Transaction): 하나의 논리적 작업 단위를 구성하는 연산들의 집합.

트랜잭션의 핵심 속성 **(ACID)**:

| 속성 | 설명 |
|------|------|
| **원자성 (Atomicity)** | 전부 실행되거나 전혀 실행되지 않아야 함 |
| **일관성 (Consistency)** | 트랜잭션 전후로 DB의 무결성 제약이 유지되어야 함 |
| **지속성 (Durability)** | 성공적으로 완료된 트랜잭션의 결과는 장애 후에도 보존되어야 함 |

> 책에서는 원자성, 일관성, 지속성을 명시적으로 언급한다. 고립성(Isolation)은 트랜잭션 관리 챕터에서 다룬다.

- **회복 관리자(Recovery Manager)**: 장애 발생 시 트랜잭션 이전 상태로 DB 복원
- **동시성 제어 관리자(Concurrency-Control Manager)**: 동시에 실행되는 트랜잭션들 간의 충돌을 방지하여 일관성 유지

---

## 1.7 데이터베이스 및 애플리케이션 아키텍처

### 집중식 vs. 분산 아키텍처
- **집중식(Centralized)**: 단일 서버 또는 공유 메모리 멀티프로세서
- **병렬(Parallel)**: 클러스터 구성으로 대용량·고성능 처리
- **분산(Distributed)**: 지리적으로 분산된 여러 머신에 걸쳐 데이터 저장·처리

### 애플리케이션 아키텍처

```
2-Tier Architecture:
[클라이언트 앱] ←─ 네트워크 ─→ [DB 서버]

3-Tier Architecture:
[클라이언트] ←→ [애플리케이션 서버] ←→ [DB 서버]
  (웹 브라우저/  (비즈니스 로직 포함)
   모바일 앱)
```

- **2-tier**: 클라이언트가 직접 DB 서버에 쿼리 전송. 단순하지만 보안·성능 취약
- **3-tier**: 비즈니스 로직이 애플리케이션 서버에 집중. 보안과 성능 모두 향상. 현대 웹/모바일 앱의 표준

---

## 1.8 데이터베이스 사용자와 관리자

### 사용자 유형

| 유형 | 설명 |
|------|------|
| **순수 사용자(Naïve Users)** | 웹/모바일 폼 인터페이스를 통해 미리 정의된 방식으로 DB와 상호작용 |
| **애플리케이션 프로그래머** | DB를 사용하는 애플리케이션을 작성하는 전문가 |
| **정교한 사용자(Sophisticated Users)** | 쿼리 언어나 데이터 분석 도구를 직접 사용하는 분석가 |

### DBA(Database Administrator)의 역할
- **스키마 정의**: DDL로 초기 DB 스키마 생성
- **저장 구조·접근 방법 정의**: 물리적 조직 및 인덱스 파라미터 설정
- **스키마·물리적 구조 변경**: 요구사항 변화 또는 성능 개선을 위한 수정
- **데이터 접근 권한 부여**: 사용자별 접근 제어
- **일상적 유지보수**: 정기 백업, 디스크 공간 관리, 성능 모니터링

---

## 1.9 데이터베이스 시스템의 역사 (History of Database Systems)

| 시대 | 주요 사건 |
|------|----------|
| **1950s~초기 1960s** | 자기 테이프(magnetic tape) 기반 데이터 처리. 순차 접근만 가능 |
| **1960s 후반~1970s** | 하드 디스크 보급으로 직접 접근(direct access) 가능. IMS(계층형), CODASYL 네트워크 모델 등장 |
| **1970s** | E.F. Codd의 관계형 모델 제안(1970). System R(IBM), INGRES(UC Berkeley) 등 프로토타입 개발 |
| **1980s** | 관계형 DBMS 상용화. SQL 표준화. DB2(IBM), Oracle 등 상용 제품 등장 |
| **1990s** | 병렬·분산 DB 발전. 데이터 웨어하우스, 의사결정 지원 시스템. 웹 애플리케이션과 DB 연동 |
| **2000s** | XML, 반구조적 데이터 처리. 오픈소스 DB(MySQL, PostgreSQL) 성장. 대용량 웹 데이터로 인한 NoSQL 등장 |
| **2010s~현재** | 클라우드 기반 DB 서비스(AWS RDS, Azure SQL). 인메모리 DB. NewSQL. 빅데이터·분석 플랫폼. AI/ML 연동 |

> **핵심 이정표**: Codd(1970)의 논문 *"A Relational Model for Large Shared Data Banks"*는 관계형 모델을 도입한 랜드마크 논문이다.

---

## 1.10 요약

- DBMS는 데이터와 그에 대한 접근 프로그램의 집합. 대용량 정보의 편리하고 효율적인 저장·검색이 목표
- 파일 처리 시스템의 한계(중복, 고립, 무결성, 원자성, 동시성, 보안)가 DBMS 발전을 이끌었다
- **데이터 추상화** 3계층(물리/논리/뷰)을 통해 복잡성을 은닉
- 데이터 모델: 관계형, E-R, 반구조적, 객체 기반
- **DDL**: 스키마 정의 / **DML**: 데이터 조작 (SQL은 선언적 DML)
- DB 설계: 개념적 설계(E-R) → 논리적 설계 → 물리적 설계
- DB 엔진 3대 컴포넌트: 저장 관리자, 쿼리 프로세서, 트랜잭션 관리자
- ACID 중 원자성·일관성·지속성이 트랜잭션의 핵심 요구사항
- 현대 앱은 3-tier 아키텍처 사용 (클라이언트 - 앱 서버 - DB 서버)

---

## 핵심 용어 정리

| 용어 | 설명 |
|------|------|
| DBMS | 데이터베이스 관리 시스템 |
| Schema | DB의 구조적 설계 |
| Instance | 특정 시점의 실제 데이터 |
| Physical Data Independence | 물리 스키마 변경이 앱에 영향 없음 |
| DDL | 데이터 정의 언어 (스키마 명세) |
| DML | 데이터 조작 언어 (질의·수정) |
| Query Language | DML의 조회 부분 |
| Data Dictionary | 메타데이터 저장소 |
| Transaction | 논리적 작업 단위 (ACID) |
| Recovery Manager | 장애 복구 담당 컴포넌트 |
| Concurrency Control | 동시 트랜잭션 충돌 방지 |
| 2-tier / 3-tier | 애플리케이션 아키텍처 패턴 |
| DBA | 데이터베이스 관리자 |
| NoSQL | 비관계형 DB (2000년대 등장) |
