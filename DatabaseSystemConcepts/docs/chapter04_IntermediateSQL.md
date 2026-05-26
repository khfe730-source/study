# Chapter 4: Intermediate SQL

## 개요

3장의 기본 SQL에서 한 단계 더 나아가 조인 표현식, 뷰, 트랜잭션, 무결성 제약, 타입 시스템, 권한 관리를 다룬다. 실무에서 스키마 설계와 애플리케이션 개발에 직접적으로 쓰이는 기능들이다.

---

## 4.1 조인 표현식 (Join Expressions)

### Natural Join

```sql
SELECT name, course_id
FROM student NATURAL JOIN takes;
```

두 릴레이션에서 **동일한 이름**을 가진 속성에 대해 자동으로 동등 조인(equi-join)을 수행하고, 결과에서 중복 속성을 제거한다.

> **위험성**: 의도치 않은 속성이 조인 조건에 포함될 수 있다. 예: `student NATURAL JOIN takes NATURAL JOIN course`에서 `student`와 `course` 모두 `dept_name`을 가지므로 `dept_name`까지 동등 조건으로 포함되어 의도치 않은 필터링이 발생.

```sql
-- 안전한 대안: USING으로 조인 속성 명시
SELECT name, title
FROM (student NATURAL JOIN takes) JOIN course USING (course_id);
```

### JOIN ... ON

```sql
SELECT name, title
FROM student JOIN takes ON student.ID = takes.ID
             JOIN course ON takes.course_id = course.course_id;
```

`ON`은 임의의 조인 조건을 지정한다. `WHERE`와 의미적으로 동등하지만, `OUTER JOIN`에서는 동작이 다르다.

### Outer Join (외부 조인)

내부 조인(inner join)은 조인 조건을 만족하는 튜플만 결과에 포함. 외부 조인은 한쪽 또는 양쪽에서 매칭되지 않는 튜플도 NULL로 채워서 포함.

```
student           takes
ID  name         ID   course_id
1   Alice        1    CS-101
2   Bob          3    MA-201
3   Charlie

--- LEFT OUTER JOIN ---
student LEFT OUTER JOIN takes ON student.ID = takes.ID

ID  name     course_id
1   Alice    CS-101
2   Bob      NULL      ← 매칭 없음, NULL로 패딩
3   Charlie  MA-201

--- RIGHT OUTER JOIN ---
ID    name     course_id
1     Alice    CS-101
NULL  NULL     MA-201    ← student에 ID=3 없는 경우 가정
3     Charlie  MA-201

--- FULL OUTER JOIN ---
양쪽 모두 포함
```

```sql
-- 수강 신청이 없는 학생 포함
SELECT ID, name, course_id
FROM student NATURAL LEFT OUTER JOIN takes;

-- FULL OUTER JOIN: 어느 쪽에도 매칭이 없는 경우 포함
SELECT *
FROM (SELECT * FROM student WHERE dept_name = 'Comp. Sci.')
FULL OUTER JOIN
(SELECT * FROM takes WHERE semester = 'Spring' AND year = 2017)
USING (ID);
```

### Inner Join (명시적)

```sql
SELECT * FROM student INNER JOIN takes USING (ID);  -- INNER는 기본값
```

`JOIN`만 쓰면 기본이 `INNER JOIN`이다.

---

## 4.2 뷰 (Views)

### CREATE VIEW

```sql
CREATE VIEW faculty AS
    SELECT ID, name, dept_name FROM instructor;
```

뷰는 실제 데이터를 저장하지 않는다. 쿼리 실행 시마다 뷰 정의를 전개(expand)하여 처리.

```sql
-- 뷰 활용
SELECT name FROM faculty WHERE dept_name = 'Biology';
-- 실제 실행: SELECT name FROM instructor WHERE dept_name = 'Biology';
```

### 뷰의 뷰 (View of Views)

```sql
CREATE VIEW physics_fall_2017 AS
    SELECT course.course_id, sec_id, building, room_number
    FROM course, section
    WHERE course.course_id = section.course_id
      AND course.dept_name = 'Physics'
      AND section.semester = 'Fall'
      AND section.year = 2017;

CREATE VIEW physics_fall_2017_watson AS
    SELECT course_id, room_number
    FROM physics_fall_2017
    WHERE building = 'Watson';
```

### 실체화 뷰 (Materialized Views)

```sql
CREATE MATERIALIZED VIEW dept_avg_salary AS
    SELECT dept_name, AVG(salary) AS avg_salary
    FROM instructor
    GROUP BY dept_name;
```

일반 뷰와 달리 결과를 실제로 저장. 기저 릴레이션 변경 시 **점진적 유지(incremental maintenance)** 또는 재계산이 필요. 복잡한 집계 쿼리의 성능을 크게 향상시킨다.

### 뷰를 통한 갱신 (Updatable Views)

```sql
-- 갱신 가능한 뷰 예시
CREATE VIEW instructor_info AS
    SELECT ID, name, building
    FROM instructor, department
    WHERE instructor.dept_name = department.dept_name;

INSERT INTO instructor_info VALUES ('69987', 'White', 'Taylor');
-- 문제: building은 department에서 오는데, dept_name을 모름 → 어디에 INSERT?
```

뷰 업데이트가 가능하려면:
1. `FROM`에 릴레이션 하나만 있어야 함
2. `SELECT`에 집계/표현식 없어야 함
3. `SELECT`에 없는 속성은 NULL 허용이어야 함
4. `DISTINCT` 없어야 함
5. `GROUP BY`/`HAVING` 없어야 함

```sql
-- WITH CHECK OPTION: 뷰 조건을 위반하는 갱신 거부
CREATE VIEW history_instructors AS
    SELECT * FROM instructor WHERE dept_name = 'History'
    WITH CHECK OPTION;

-- 아래 INSERT는 거부됨 (dept_name = 'Music' ≠ 'History')
INSERT INTO history_instructors VALUES ('25566', 'Brown', 'Music', 100000);
```

---

## 4.3 트랜잭션 (Transactions)

```sql
BEGIN;  -- 또는 START TRANSACTION (구현체에 따라 다름)

UPDATE account SET balance = balance - 100 WHERE acct_no = 'A-101';
UPDATE account SET balance = balance + 100 WHERE acct_no = 'A-201';

COMMIT;   -- 영구 반영
-- 또는
ROLLBACK; -- 모든 변경 취소
```

SQL 표준은 트랜잭션을 명시적으로 시작하지 않으며, 각 SQL 문이 암묵적으로 트랜잭션을 시작한다. `COMMIT` 또는 `ROLLBACK`으로 종료.

많은 DBMS에서 **자동 커밋(auto-commit)** 모드가 기본이다. JDBC에서는 `connection.setAutoCommit(false)`로 비활성화.

---

## 4.4 무결성 제약 (Integrity Constraints)

### 단일 릴레이션 제약

```sql
CREATE TABLE instructor (
    ID        varchar(5)   PRIMARY KEY,
    name      varchar(20)  NOT NULL,
    dept_name varchar(20),
    salary    numeric(8,2) CHECK (salary > 29000),  -- 임의 술어
    UNIQUE    (name, dept_name)                      -- 복합 유니크
);
```

- **NOT NULL**: NULL 불허
- **UNIQUE**: 지정된 속성의 조합이 모든 튜플에서 고유해야 함. NULL은 UNIQUE를 위반하지 않는다 (NULL ≠ NULL).
- **CHECK(P)**: 임의의 술어 P가 모든 튜플에서 참이어야 함

```sql
-- CHECK에 서브쿼리 (일부 DB만 지원)
CHECK (time_slot_id IN (SELECT time_slot_id FROM time_slot))
```

### 참조 무결성 (Referential Integrity)

```sql
CREATE TABLE course (
    course_id varchar(8) PRIMARY KEY,
    dept_name varchar(20),
    FOREIGN KEY (dept_name) REFERENCES department(dept_name)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

외래 키 위반 시 처리 옵션:

| 옵션 | ON DELETE | ON UPDATE |
|------|-----------|-----------|
| `NO ACTION` (기본) | 삭제 거부 | 갱신 거부 |
| `RESTRICT` | 즉시 거부 (DEFERRED도 없음) | 즉시 거부 |
| `CASCADE` | 참조 튜플도 함께 삭제 | 참조 튜플도 함께 갱신 |
| `SET NULL` | 참조 속성을 NULL로 변경 | 참조 속성을 NULL로 변경 |
| `SET DEFAULT` | 기본값으로 변경 | 기본값으로 변경 |

> **CASCADE 주의**: 연쇄 삭제가 의도치 않게 많은 데이터를 지울 수 있다.

### 제약 이름 지정

```sql
CREATE TABLE instructor (
    salary numeric(8,2),
    CONSTRAINT salary_check CHECK (salary > 29000)
);

-- 나중에 제약 제거
ALTER TABLE instructor DROP CONSTRAINT salary_check;
```

### 지연 제약 검사 (Deferred Constraint Checking)

```sql
SET CONSTRAINTS salary_check DEFERRED;
-- 이후 DML 수행
COMMIT;  -- 커밋 시점에 제약 검사
```

순환 참조나 초기 데이터 로딩에 유용. 기본 선언:

```sql
CONSTRAINT salary_check CHECK (...) INITIALLY DEFERRED
```

### 어설션 (Assertions)

```sql
CREATE ASSERTION credits_earned_constraint CHECK (
    NOT EXISTS (
        SELECT ID FROM student
        WHERE total_cred <> (
            SELECT COALESCE(SUM(credits), 0)
            FROM takes NATURAL JOIN course
            WHERE student.ID = takes.ID
              AND grade IS NOT NULL AND grade <> 'F'
        )
    )
);
```

어설션은 DB 전체에 적용되는 제약. 구현 비용이 크기 때문에 많은 DBMS에서 지원하지 않는다. PostgreSQL, Oracle 미지원, SQL:1999 표준에는 있음.

---

## 4.5 SQL 데이터 타입과 스키마

### 날짜 및 시간 타입

```sql
DATE  '2018-04-25'          -- year-month-day
TIME  '09:30:00'            -- hour:minute:second
TIMESTAMP '2018-04-25 10:29:01.45'

-- 현재 시각
CURRENT_DATE
CURRENT_TIME
CURRENT_TIMESTAMP
LOCALTIME
LOCALTIMESTAMP  -- 시간대 정보 없이

-- 시간 추출
EXTRACT(year FROM CURRENT_TIMESTAMP)
EXTRACT(month FROM order_date)
```

```sql
-- 날짜/시간 형변환
CAST('2018-04-25' AS DATE)
```

### COALESCE

```sql
-- NULL 대체값
SELECT ID, COALESCE(salary, 0) AS salary FROM instructor;
-- COALESCE(a, b, c): 첫 번째 비-NULL 값 반환
```

### 대용량 객체 (LOB, Large Object)

```sql
book_review    CLOB(10KB)   -- Character LOB: 긴 텍스트
image          BLOB(10MB)   -- Binary LOB: 이미지, 영상 등
movie          BLOB(2GB)
```

LOB는 DB에 직접 저장되거나 외부 파일 참조 방식으로 처리. 조회 시 전체를 메모리에 올리지 않고 **로케이터(locator)** 를 통해 스트리밍.

### 사용자 정의 타입 (User-Defined Types)

```sql
-- 구별 타입 (Distinct Type)
CREATE TYPE Dollars AS NUMERIC(12, 2) FINAL;
CREATE TYPE Pounds   AS NUMERIC(12, 2) FINAL;

CREATE TABLE department (
    budget Dollars
);

-- Dollars와 Pounds는 타입이 달라 직접 비교 불가 → 타입 안전성
CAST(department.budget TO NUMERIC(12,2))  -- 필요시 변환
```

```sql
-- 도메인 (Domain): 타입 + 제약
CREATE DOMAIN DDollars AS NUMERIC(12, 2) NOT NULL;
CREATE DOMAIN YearlySalary AS NUMERIC(8,2)
    CONSTRAINT salary_value_test CHECK (VALUE >= 29000.00);
```

`DOMAIN`은 `NOT NULL`이나 `CHECK`를 내포할 수 있어 스키마 재사용성이 높다. `TYPE`은 더 강한 타입 안전성을 제공.

### GENERATED (Identity) 컬럼

```sql
CREATE TABLE student_id_seq (
    ID NUMERIC GENERATED ALWAYS AS IDENTITY,
    name varchar(20)
);
-- INSERT 시 ID를 명시하지 않으면 자동 생성
```

---

## 4.6 권한 관리 (Authorization)

### 권한 종류

| 권한 | 의미 |
|------|------|
| `SELECT` | 데이터 조회 |
| `INSERT` | 행 삽입 |
| `UPDATE` | 값 수정 |
| `DELETE` | 행 삭제 |
| `REFERENCES` | 외래 키 생성 권한 |
| `USAGE` | 도메인 등 리소스 사용 |
| `ALL PRIVILEGES` | 모든 권한 |

### GRANT / REVOKE

```sql
-- 권한 부여
GRANT SELECT ON department TO Amit, Satoshi;
GRANT UPDATE (budget) ON department TO Amit;  -- 특정 속성만

-- WITH GRANT OPTION: 다른 사용자에게 재위임 가능
GRANT SELECT ON department TO Amit WITH GRANT OPTION;

-- 권한 회수
REVOKE SELECT ON department FROM Amit, Satoshi;

-- CASCADE: 재위임한 권한도 연쇄 회수
REVOKE SELECT ON department FROM Amit CASCADE;

-- RESTRICT: 재위임한 권한이 있으면 회수 거부
REVOKE SELECT ON department FROM Amit RESTRICT;
```

### 롤(Role)

```sql
-- 롤 생성 및 권한 할당
CREATE ROLE instructor;
GRANT SELECT ON takes TO instructor;

-- 사용자에게 롤 부여
GRANT instructor TO Amit;

-- 롤 간 계층
CREATE ROLE teaching_assistant;
GRANT instructor TO teaching_assistant;  -- instructor의 권한을 TA도 가짐

-- 롤 사용 설정
SET ROLE instructor;
```

### 권한 그래프 (Authorization Graph)

```
DBA
 ├─ GRANT SELECT → User A (WITH GRANT OPTION)
 │        └─ GRANT SELECT → User B
 │                 └─ GRANT SELECT → User C
 └─ GRANT SELECT → User B (직접)
```

`REVOKE ... CASCADE` 시: A로부터 파생된 B의 권한은 회수되지만, DBA가 직접 준 B의 권한은 유지된다. **권한 그래프에서 DBA 노드까지 경로가 남아있으면 권한 유지**.

### 뷰를 통한 권한 제어

```sql
-- 특정 학과 데이터만 접근 가능한 뷰
CREATE VIEW geo_instructor AS
    SELECT * FROM instructor WHERE dept_name = 'Geology';

GRANT SELECT ON geo_instructor TO geo_staff;
-- geo_staff는 Geology 학과 데이터만 볼 수 있고, instructor 전체에는 접근 불가
```

---

## 핵심 정리

### 조인 종류 요약

```
INNER JOIN  = 매칭되는 행만
LEFT OUTER  = 왼쪽 테이블 모든 행 + 오른쪽 NULL 패딩
RIGHT OUTER = 오른쪽 테이블 모든 행 + 왼쪽 NULL 패딩
FULL OUTER  = 양쪽 모든 행, 미매칭은 NULL 패딩
NATURAL     = 동명 속성으로 자동 조인 (위험할 수 있음)
JOIN USING  = 명시된 속성으로 조인
JOIN ON     = 임의 조건 조인
CROSS JOIN  = 카테시안 곱
```

### 뷰 vs 실체화 뷰

```
                뷰(View)          실체화 뷰(Materialized View)
저장 방식      쿼리 정의만 저장   결과 데이터 저장
실행 시점      매 조회마다 실행   사전 계산, 갱신 시 재계산
성능           복잡 쿼리 시 느림  집계 쿼리 빠름
최신성         항상 최신          갱신 주기에 따라 지연
```

### 외래 키 옵션 선택 지침

```
ON DELETE CASCADE   → 부모 삭제 시 자식도 삭제해야 하는 경우 (ex: 주문-주문상세)
ON DELETE SET NULL  → 참조 관계를 끊고 자식은 남겨야 하는 경우 (ex: 직원-부서)
ON DELETE RESTRICT  → 참조 중인 부모는 삭제 불가 (ex: 계정-잔액)
```

### SQL 타입 계층

```
내장 타입(Built-in)
├── 숫자: INTEGER, NUMERIC, REAL, FLOAT
├── 문자: CHAR, VARCHAR, CLOB
├── 이진: BIT, BLOB
└── 날짜/시간: DATE, TIME, TIMESTAMP

사용자 정의
├── CREATE TYPE  (강한 타입 검사, CAST 필요)
└── CREATE DOMAIN (제약 포함 타입 별칭)
```
