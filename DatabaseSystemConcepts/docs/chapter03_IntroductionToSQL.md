# Chapter 3: Introduction to SQL

## 개요

SQL(Structured Query Language)은 관계형 데이터베이스의 표준 언어로, IBM의 System R 프로젝트에서 파생된 SEQUEL이 그 기원이다. DDL(Data Definition Language), DML(Data Manipulation Language), 무결성 제약, 뷰 정의, 트랜잭션 제어, 권한 관리를 모두 포괄한다. 현대 SQL 표준(SQL-92, SQL:1999, SQL:2003, SQL:2011, SQL:2016)은 지속적으로 확장되고 있다.

---

## 3.1 SQL 데이터 정의 (SQL Data Definition)

### 기본 데이터 타입

| 타입 | 설명 |
|------|------|
| `char(n)` | 고정 길이 문자열. 부족한 자리는 공백으로 패딩 |
| `varchar(n)` | 가변 길이 문자열. 최대 n자 |
| `int` / `integer` | 4바이트 정수 |
| `smallint` | 2바이트 정수 |
| `numeric(p, d)` | 고정 소수점. 전체 p자리, 소수점 이하 d자리 |
| `real`, `double precision` | 부동 소수점 (구현체 의존 정밀도) |
| `float(n)` | 최소 n비트 유효 자리의 부동 소수점 |

> **주의**: `char`와 `varchar`의 동등 비교는 구현체마다 다르게 처리할 수 있다. `'Avi' = 'Avi '`가 true일 수도 있고 false일 수도 있다. 이식성을 위해 `varchar`를 선호한다.

### CREATE TABLE

```sql
CREATE TABLE instructor (
    ID          varchar(5),
    name        varchar(20)  NOT NULL,
    dept_name   varchar(20),
    salary      numeric(8, 2),
    PRIMARY KEY (ID),
    FOREIGN KEY (dept_name) REFERENCES department(dept_name)
);
```

- `PRIMARY KEY`: NOT NULL + UNIQUE를 암묵적으로 내포
- `FOREIGN KEY ... REFERENCES`: 참조 무결성(referential integrity) 강제
- `NOT NULL`: 해당 속성에 NULL 값 불허

### DROP TABLE / ALTER TABLE

```sql
DROP TABLE student;                          -- 테이블과 데이터 모두 삭제

ALTER TABLE instructor ADD COLUMN phone varchar(20);  -- 속성 추가
ALTER TABLE instructor DROP COLUMN phone;             -- 속성 제거
```

`DROP TABLE`은 `DELETE FROM table`과 달리 스키마 자체를 제거한다.

---

## 3.2 SQL 쿼리 기본 구조

### SELECT-FROM-WHERE

```sql
SELECT A1, A2, ..., An
FROM   r1, r2, ..., rm
WHERE  P
```

이는 관계 대수(relational algebra)로 표현하면:

```
Π_{A1,...,An}(σ_P(r1 × r2 × ... × rm))
```

`FROM`절의 복수 릴레이션은 카테시안 곱(Cartesian product)을 생성한다. 실제 실행 시 DBMS 옵티마이저가 물리적으로 곱을 만들지 않고 조인 알고리즘을 선택한다.

### DISTINCT / ALL

```sql
SELECT DISTINCT dept_name FROM instructor;   -- 중복 제거
SELECT ALL dept_name FROM instructor;        -- 중복 허용 (기본값)
```

SQL은 기본적으로 **다중 집합(multiset, bag)** 의미론을 따른다. 관계 대수의 집합 의미론과 다르다.

### AS (이름 변경, Rename)

```sql
SELECT name AS instructor_name, course_id
FROM instructor, teaches
WHERE instructor.ID = teaches.ID;

-- 릴레이션 이름 변경 (자기 조인에 필수)
SELECT T.name, S.course_id
FROM instructor AS T, teaches AS S
WHERE T.ID = S.ID;
```

### 문자열 연산

SQL은 문자열에 **작은따옴표** `'` 사용. 문자열 내 작은따옴표는 두 개 연속(`''`)으로 이스케이프.

```sql
-- LIKE 패턴 매칭
-- %: 임의 길이 부분 문자열
-- _: 임의 단일 문자
SELECT name FROM instructor WHERE name LIKE '%dar%';
SELECT name FROM instructor WHERE name LIKE '_ _ _';   -- 정확히 3자

-- ESCAPE 문자 지정
SELECT name FROM department WHERE name LIKE '100\%' ESCAPE '\';
```

대소문자 구분은 구현체마다 다르다 (MySQL은 기본적으로 대소문자 무시, PostgreSQL은 구분).

### ORDER BY

```sql
SELECT name, salary
FROM instructor
WHERE dept_name = 'Physics'
ORDER BY salary DESC, name ASC;  -- 1차 정렬: salary 내림차순, 2차: name 오름차순
```

### BETWEEN / NOT BETWEEN

```sql
SELECT name FROM instructor WHERE salary BETWEEN 90000 AND 100000;
-- 동등: salary >= 90000 AND salary <= 100000
```

### 튜플 비교

```sql
SELECT name, course_id
FROM instructor, teaches
WHERE (instructor.ID, dept_name) = (teaches.ID, 'Biology');
```

---

## 3.3 집합 연산 (Set Operations)

SQL의 집합 연산은 기본적으로 **중복을 제거**한다. `ALL` 키워드로 다중 집합 의미론 사용 가능.

```sql
-- UNION: 합집합
(SELECT course_id FROM section WHERE semester = 'Fall' AND year = 2017)
UNION
(SELECT course_id FROM section WHERE semester = 'Spring' AND year = 2018);

-- INTERSECT: 교집합
(SELECT course_id FROM section WHERE semester = 'Fall' AND year = 2017)
INTERSECT
(SELECT course_id FROM section WHERE semester = 'Spring' AND year = 2018);

-- EXCEPT: 차집합 (일부 DB에서는 MINUS)
(SELECT course_id FROM section WHERE semester = 'Fall' AND year = 2017)
EXCEPT
(SELECT course_id FROM section WHERE semester = 'Spring' AND year = 2018);
```

다중 집합(multiset) 버전:

```sql
UNION ALL     -- 중복 유지
INTERSECT ALL -- 공통 원소는 min(count1, count2)개 유지
EXCEPT ALL    -- count1 - count2개 유지 (음수면 0)
```

---

## 3.4 NULL 값

NULL은 "값 없음" 또는 "알 수 없음"을 나타낸다. SQL은 **3값 논리(three-valued logic)**: `true`, `false`, `unknown`.

### NULL 연산 규칙

```
NULL + 5     = NULL     -- 산술 연산 결과는 NULL
NULL = NULL  = unknown  -- 비교 연산 결과는 unknown (true가 아님!)
NULL AND true  = unknown
NULL AND false = false  -- false는 어떤 값과 AND해도 false
NULL OR true   = true   -- true는 어떤 값과 OR해도 true
NULL OR false  = unknown
NOT unknown    = unknown
```

`WHERE` 절은 결과가 `true`인 튜플만 포함시킨다. `unknown`은 탈락.

### IS NULL / IS NOT NULL

```sql
SELECT name FROM instructor WHERE salary IS NULL;
SELECT name FROM instructor WHERE salary IS NOT NULL;
```

> **중요**: `salary = NULL`은 항상 `unknown`을 반환하므로 어떤 튜플도 선택되지 않는다. 반드시 `IS NULL`을 사용해야 한다.

---

## 3.5 집계 함수 (Aggregate Functions)

### 기본 집계

```sql
SELECT AVG(salary) AS avg_salary FROM instructor WHERE dept_name = 'Comp. Sci.';
SELECT COUNT(DISTINCT ID) FROM teaches WHERE semester = 'Spring' AND year = 2018;
SELECT COUNT(*) FROM course;   -- NULL 포함 전체 행 수
```

| 함수 | 설명 |
|------|------|
| `AVG(col)` | 평균. NULL 무시 |
| `MIN(col)` | 최솟값. NULL 무시 |
| `MAX(col)` | 최댓값. NULL 무시 |
| `SUM(col)` | 합계. NULL 무시 |
| `COUNT(col)` | NULL이 아닌 값의 수 |
| `COUNT(*)` | 튜플 수 (NULL 포함) |

### GROUP BY

```sql
SELECT dept_name, AVG(salary) AS avg_salary
FROM instructor
GROUP BY dept_name;
```

`GROUP BY`에 명시된 속성만 `SELECT`에서 집계 없이 나타날 수 있다. 그 외 속성은 반드시 집계 함수로 감싸야 한다.

```sql
-- 오류: name은 GROUP BY에도 없고 집계 함수도 아님
SELECT dept_name, ID, AVG(salary) FROM instructor GROUP BY dept_name;
```

### HAVING

```sql
SELECT dept_name, AVG(salary) AS avg_salary
FROM instructor
GROUP BY dept_name
HAVING AVG(salary) > 42000;
```

`HAVING`은 **그룹 형성 후** 필터링. `WHERE`는 **그룹 형성 전** 개별 튜플 필터링.

### 실행 순서 개념도

```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
```

NULL 값을 가진 튜플은 집계 함수에서 무시된다. 단, `COUNT(*)`는 예외.

---

## 3.6 중첩 서브쿼리 (Nested Subqueries)

### IN / NOT IN

```sql
-- Fall 2017과 Spring 2018 모두 개설된 과목
SELECT DISTINCT course_id
FROM section
WHERE semester = 'Fall' AND year = 2017
  AND course_id IN (
      SELECT course_id FROM section
      WHERE semester = 'Spring' AND year = 2018
  );
```

### SOME / ALL

```sql
-- Biology 학과의 어떤 교수보다 급여가 높은 교수
SELECT name FROM instructor
WHERE salary > SOME (
    SELECT salary FROM instructor WHERE dept_name = 'Biology'
);

-- 모든 Biology 교수보다 급여가 높은 교수
SELECT name FROM instructor
WHERE salary > ALL (
    SELECT salary FROM instructor WHERE dept_name = 'Biology'
);
```

`= SOME`은 `IN`과 동등. `<> ALL`은 `NOT IN`과 동등.

### EXISTS / NOT EXISTS

```sql
-- Fall 2017과 Spring 2018 모두 개설된 과목 (EXISTS 버전)
SELECT course_id
FROM section AS S
WHERE semester = 'Fall' AND year = 2017
  AND EXISTS (
      SELECT * FROM section AS T
      WHERE T.semester = 'Spring' AND T.year = 2018
          AND S.course_id = T.course_id
  );
```

`EXISTS`는 서브쿼리 결과가 비어 있지 않으면 `true`. 외부 쿼리의 속성을 참조하는 **상관 서브쿼리(correlated subquery)**.

```sql
-- 모든 Biology 과목을 수강한 학생
SELECT DISTINCT S.ID, S.name
FROM student AS S
WHERE NOT EXISTS (
    (SELECT course_id FROM course WHERE dept_name = 'Biology')
    EXCEPT
    (SELECT T.course_id FROM takes AS T WHERE T.ID = S.ID)
);
```

### UNIQUE

```sql
-- 2017년에 최대 한 번 개설된 과목
SELECT T.course_id FROM course AS T
WHERE UNIQUE (
    SELECT R.course_id FROM section AS R
    WHERE T.course_id = R.course_id AND R.year = 2017
);
```

서브쿼리에 중복 튜플이 없으면 `true`. (NULL 처리 주의: NULL = NULL이 unknown이므로 NULL이 있는 경우 UNIQUE는 true를 반환할 수 있다)

### FROM 절 서브쿼리 (Derived Relation)

```sql
SELECT dept_name, avg_salary
FROM (
    SELECT dept_name, AVG(salary) AS avg_salary
    FROM instructor
    GROUP BY dept_name
) AS dept_avg
WHERE avg_salary > 42000;
```

FROM 절 서브쿼리는 반드시 **별칭(alias)** 이 있어야 한다.

### WITH 절 (CTE, Common Table Expression)

```sql
WITH dept_total(dept_name, value) AS (
    SELECT dept_name, SUM(salary)
    FROM instructor
    GROUP BY dept_name
),
dept_total_avg(value) AS (
    SELECT AVG(value) FROM dept_total
)
SELECT dept_name
FROM dept_total, dept_total_avg
WHERE dept_total.value >= dept_total_avg.value;
```

`WITH`는 복잡한 쿼리를 단계적으로 분해하여 가독성을 높인다. 동일 쿼리 내에서 여러 번 재사용 가능 (서브쿼리보다 효율적일 수 있음).

### 스칼라 서브쿼리 (Scalar Subquery)

```sql
-- SELECT 절에서 스칼라 서브쿼리
SELECT dept_name,
       (SELECT COUNT(*) FROM instructor AS I
        WHERE I.dept_name = D.dept_name) AS num_instructors
FROM department AS D;
```

정확히 하나의 값을 반환해야 한다. 복수 행이 반환되면 런타임 오류.

---

## 3.7 데이터 수정 (Modification of the Database)

### DELETE

```sql
DELETE FROM instructor;                          -- 전체 삭제

DELETE FROM instructor WHERE dept_name = 'Finance';

-- 서브쿼리 활용: 평균 급여 미만 교수 삭제
DELETE FROM instructor
WHERE salary < (SELECT AVG(salary) FROM instructor);
```

> **주의**: DELETE는 투 패스로 처리된다. 서브쿼리가 먼저 전체 평가되고 나서 삭제가 수행된다.

### INSERT

```sql
-- 단순 삽입
INSERT INTO course VALUES ('CS-437', 'Database Systems', 'Comp. Sci.', 4);

-- 일부 속성만 지정 (나머지는 NULL)
INSERT INTO course (course_id, title, dept_name, credits)
VALUES ('CS-437', 'Database Systems', 'Comp. Sci.', 4);

-- SELECT 결과로 삽입
INSERT INTO instructor
    SELECT ID, name, dept_name, 18000
    FROM student
    WHERE dept_name = 'Music' AND total_cred > 144;
```

### UPDATE

```sql
-- 단순 업데이트
UPDATE instructor SET salary = salary * 1.05;

-- 조건부 업데이트
UPDATE instructor
SET salary = salary * 1.05
WHERE salary < 70000;

-- CASE를 이용한 조건부 업데이트
UPDATE instructor
SET salary = CASE
    WHEN salary <= 100000 THEN salary * 1.05
    ELSE salary * 1.03
END;
```

`CASE` 표현식:

```sql
CASE
    WHEN pred1 THEN result1
    WHEN pred2 THEN result2
    ...
    ELSE result0
END
```

`CASE`는 표현식(expression)이므로 `SELECT`, `WHERE`, `UPDATE SET` 등 어디서든 사용 가능.

### 스칼라 서브쿼리를 이용한 UPDATE

```sql
-- 학생 total_cred를 수강 완료 과목의 학점 합으로 갱신
UPDATE student S
SET total_cred = (
    SELECT COALESCE(SUM(credits), 0)
    FROM takes NATURAL JOIN course
    WHERE S.ID = takes.ID
      AND takes.grade <> 'F'
      AND takes.grade IS NOT NULL
);
```

---

## 핵심 정리

### SQL 다중 집합 의미론

SQL은 집합(set)이 아닌 **다중 집합(multiset/bag)** 으로 작동한다. 중복 제거는 `DISTINCT`를 명시해야 한다. 이는 `COUNT(*)` 같은 집계에서 의미가 있다.

### 쿼리 평가 순서 (개념적)

```
1. FROM     -- 카테시안 곱 또는 조인
2. WHERE    -- 튜플 필터링
3. GROUP BY -- 그룹화
4. HAVING   -- 그룹 필터링
5. SELECT   -- 속성 선택 / 집계 계산
6. DISTINCT -- 중복 제거 (선택적)
7. ORDER BY -- 정렬
```

### NULL 처리 요약

```
산술: NULL op x = NULL
비교: NULL comp x = unknown
논리: unknown은 WHERE에서 제거됨
집계: NULL은 무시됨 (COUNT(*) 제외)
```

### 서브쿼리 종류 비교

| 종류 | 위치 | 반환 | 예시 |
|------|------|------|------|
| 스칼라 | SELECT/WHERE | 단일 값 | `(SELECT MAX(salary)...)` |
| 집합 | WHERE (IN/EXISTS) | 집합 | `WHERE id IN (SELECT ...)` |
| 파생 릴레이션 | FROM | 테이블 | `FROM (SELECT ...) AS t` |
| CTE | WITH | 테이블 | `WITH t AS (SELECT ...)` |
| 상관 | 어디서나 | 외부 참조 | `EXISTS (SELECT ... WHERE outer.x = inner.y)` |
