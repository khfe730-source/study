# Chapter 5: Advanced SQL

## 개요

SQL을 프로그래밍 언어에서 사용하는 방법(JDBC/ODBC/Embedded SQL), DB 서버 측 로직(함수, 프로시저, 트리거), 그리고 분석용 고급 쿼리(재귀, 윈도우 함수, OLAP)를 다룬다.

---

## 5.1 프로그래밍 언어에서 SQL 접근

### JDBC (Java Database Connectivity)

JDBC는 자바 프로그램이 DBMS에 독립적으로 접근할 수 있는 API다.

```java
// 1. 드라이버 로드 및 연결
import java.sql.*;

public class JdbcExample {
    public static void main(String[] args) throws Exception {
        // 드라이버 자동 로드 (JDBC 4.0+)
        String url = "jdbc:oracle:thin:@db.yale.edu:1521:univdb";
        Connection conn = DriverManager.getConnection(url, "user", "password");
        conn.setAutoCommit(false);  // 트랜잭션 수동 관리
        
        // 2. Statement 실행
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery(
            "SELECT dept_name, AVG(salary) FROM instructor GROUP BY dept_name"
        );
        
        while (rs.next()) {
            System.out.println(rs.getString("dept_name") + " " + rs.getFloat(2));
        }
        rs.close();
        stmt.close();
        
        // 3. PreparedStatement (SQL Injection 방지 + 성능)
        PreparedStatement pstmt = conn.prepareStatement(
            "INSERT INTO instructor VALUES(?, ?, ?, ?)"
        );
        pstmt.setString(1, "88877");
        pstmt.setString(2, "Perry");
        pstmt.setString(3, "Finance");
        pstmt.setInt(4, 125000);
        pstmt.executeUpdate();
        
        conn.commit();
        conn.close();
    }
}
```

**PreparedStatement vs Statement:**

```
Statement         → SQL 문자열을 매번 파싱/컴파일
                    SQL Injection 취약
PreparedStatement → 한 번 컴파일 후 파라미터만 바인딩
                    안전, 반복 실행 시 성능 우수
```

### ResultSet 메타데이터

```java
ResultSetMetaData rsmd = rs.getMetaData();
int cols = rsmd.getColumnCount();
for (int i = 1; i <= cols; i++) {
    System.out.println(rsmd.getColumnName(i) + " " + rsmd.getColumnTypeName(i));
}

// DatabaseMetaData: DB 스키마 탐색
DatabaseMetaData dbmd = conn.getMetaData();
ResultSet tables = dbmd.getColumns(null, null, "instructor", "%");
```

### NULL 처리 in JDBC

```java
int salary = rs.getInt("salary");
if (rs.wasNull()) {
    // salary는 NULL이었음
}

// NULL 삽입
pstmt.setNull(4, java.sql.Types.NUMERIC);
```

### ODBC (Open Database Connectivity)

C/C++ 기반의 DBMS 접근 API. JDBC의 C 버전에 해당. ADO.NET(.NET Framework), pyodbc(Python) 등이 ODBC 위에서 동작.

```c
SQLHENV env;  SQLHDBC conn;  SQLHSTMT stmt;
SQLAllocHandle(SQL_HANDLE_ENV, SQL_NULL_HANDLE, &env);
SQLSetEnvAttr(env, SQL_ATTR_ODBC_VERSION, SQL_OV_ODBC3, 0);
SQLAllocHandle(SQL_HANDLE_DBC, env, &conn);
SQLConnect(conn, "DB_DSN", SQL_NTS, "user", SQL_NTS, "pass", SQL_NTS);
SQLAllocHandle(SQL_HANDLE_STMT, conn, &stmt);
SQLExecDirect(stmt, "SELECT name FROM instructor", SQL_NTS);
while (SQLFetch(stmt) == SQL_SUCCESS) {
    char name[50];
    SQLGetData(stmt, 1, SQL_C_CHAR, name, sizeof(name), NULL);
}
```

### Embedded SQL

SQL 문을 프로그램 소스 코드에 직접 내장. 전처리기(preprocessor)가 SQL을 DBMS 호출 코드로 변환.

```c
EXEC SQL INCLUDE SQLCA;   /* SQL Communication Area */

EXEC SQL BEGIN DECLARE SECTION;
    char V_ID[6], V_name[21];
    float V_salary;
EXEC SQL END DECLARE SECTION;

/* 커서 선언 및 사용 */
EXEC SQL DECLARE c CURSOR FOR
    SELECT ID, name, salary FROM instructor WHERE dept_name = :V_dept;

EXEC SQL OPEN c;
EXEC SQL FETCH c INTO :V_ID, :V_name, :V_salary;
EXEC SQL CLOSE c;
```

호스트 변수(host variable)는 `:변수명`으로 SQL 내에서 참조.

### Python DB-API

```python
import psycopg2  # PostgreSQL 드라이버

conn = psycopg2.connect(host="...", database="university", user="...", password="...")
cursor = conn.cursor()

cursor.execute("SELECT * FROM instructor WHERE dept_name = %s", ("Physics",))
for row in cursor.fetchall():
    print(row)

cursor.execute("UPDATE instructor SET salary = salary * 1.05 WHERE dept_name = %s",
               ("Physics",))
conn.commit()
conn.close()
```

---

## 5.2 함수와 프로시저 (Functions and Procedures)

DB 서버 측에서 로직을 실행. 네트워크 왕복(round-trip) 감소, 보안 강화, 코드 재사용.

### SQL 함수

```sql
-- 학과의 교수 수 반환
CREATE FUNCTION dept_count(dept_name varchar(20))
RETURNS integer
BEGIN
    DECLARE d_count integer;
    SELECT COUNT(*) INTO d_count
    FROM instructor
    WHERE instructor.dept_name = dept_count.dept_name;
    RETURN d_count;
END;

-- 사용
SELECT dept_name, budget
FROM department
WHERE dept_count(dept_name) > 12;
```

### 테이블 함수 (Table Function)

```sql
CREATE FUNCTION instructors_of(dept_name varchar(20))
RETURNS TABLE (
    ID varchar(5),
    name varchar(20),
    dept_name varchar(20),
    salary numeric(8,2)
)
RETURN TABLE (
    SELECT ID, name, dept_name, salary
    FROM instructor
    WHERE instructor.dept_name = instructors_of.dept_name
);

-- FROM 절에서 사용
SELECT * FROM TABLE(instructors_of('Music'));
```

### SQL 프로시저 (PSM, Persistent Stored Modules)

```sql
CREATE PROCEDURE dept_count_proc(
    IN   dept_name  varchar(20),
    OUT  d_count    integer
)
BEGIN
    SELECT COUNT(*) INTO d_count
    FROM instructor
    WHERE instructor.dept_name = dept_count_proc.dept_name;
END;

-- 호출
DECLARE d_count integer;
CALL dept_count_proc('Physics', d_count);
```

### PSM 제어 구조

```sql
CREATE PROCEDURE proc_example()
BEGIN
    DECLARE n INTEGER DEFAULT 0;
    DECLARE done BOOLEAN DEFAULT FALSE;
    
    -- WHILE
    WHILE n < 10 DO
        SET n = n + 1;
    END WHILE;
    
    -- REPEAT ... UNTIL
    REPEAT
        SET n = n - 1;
    UNTIL n = 0 END REPEAT;
    
    -- FOR (커서 순회)
    FOR r AS
        SELECT dept_name, budget FROM department
    DO
        -- r.dept_name, r.budget 사용
    END FOR;
    
    -- IF ... THEN ... ELSEIF ... ELSE
    IF salary > 100000 THEN
        SET bonus = salary * 0.10;
    ELSEIF salary > 50000 THEN
        SET bonus = salary * 0.05;
    ELSE
        SET bonus = 1000;
    END IF;
    
    -- CASE
    CASE grade
        WHEN 'A' THEN SET points = 4.0;
        WHEN 'B' THEN SET points = 3.0;
        ELSE          SET points = 0.0;
    END CASE;
END;
```

### 예외 처리

```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    ROLLBACK;
    -- 에러 처리
END;

-- 사용자 정의 예외
DECLARE too_many_students CONDITION;
IF student_count > 200 THEN
    SIGNAL too_many_students
        SET MESSAGE_TEXT = 'Section is full';
END IF;
```

### 외부 언어 루틴 (External Language Routines)

```sql
-- C 함수를 DB에 등록 (PostgreSQL 예시)
CREATE FUNCTION gcd(integer, integer)
RETURNS integer
AS 'gcd.so', 'gcd'
LANGUAGE C STRICT;
```

Java, Python, R 등도 지원하는 DBMS가 많다. 보안을 위해 샌드박스 환경에서 실행.

---

## 5.3 트리거 (Triggers)

트리거는 DB의 특정 이벤트(INSERT/UPDATE/DELETE) 발생 시 자동으로 실행되는 코드.

### 트리거 구조

```sql
CREATE TRIGGER trigger_name
{BEFORE | AFTER} {INSERT | UPDATE [OF col] | DELETE}
ON table_name
[REFERENCING
    {NEW | OLD} {ROW | TABLE} [AS alias]
]
[FOR EACH {ROW | STATEMENT}]
[WHEN (condition)]
BEGIN
    -- 실행 로직
END;
```

### 예시: 급여 변경 감사 로그

```sql
CREATE TRIGGER salary_audit
AFTER UPDATE OF salary ON instructor
REFERENCING OLD ROW AS old_row
            NEW ROW AS new_row
FOR EACH ROW
WHEN (new_row.salary <> old_row.salary)
BEGIN
    INSERT INTO salary_log(instructor_id, old_salary, new_salary, change_date)
    VALUES (new_row.ID, old_row.salary, new_row.salary, CURRENT_DATE);
END;
```

### 예시: 재고 보충 트리거 (AFTER INSERT)

```sql
CREATE TRIGGER reorder_trigger
AFTER UPDATE OF amount ON inventory
REFERENCING OLD ROW AS old_row
            NEW ROW AS new_row
FOR EACH ROW
WHEN (new_row.amount < new_row.min_inventory
      AND old_row.amount >= old_row.min_inventory)
BEGIN ATOMIC
    INSERT INTO orders(item_id, quantity)
    VALUES (new_row.item_id, new_row.max_inventory - new_row.amount);
END;
```

### FOR EACH ROW vs FOR EACH STATEMENT

```
FOR EACH ROW       → 영향을 받는 각 행마다 트리거 실행
                     REFERENCING NEW/OLD ROW 사용 가능
FOR EACH STATEMENT → SQL 문 전체에 대해 한 번 실행
                     REFERENCING NEW/OLD TABLE 사용 (전환 테이블, transition table)
```

### BEFORE 트리거: 데이터 정규화

```sql
CREATE TRIGGER setnull_trigger
BEFORE UPDATE OR INSERT ON instructor
REFERENCING NEW ROW AS nrow
FOR EACH ROW
BEGIN ATOMIC
    IF nrow.dept_name IS NULL THEN
        SET nrow.salary = 0;
    END IF;
END;
```

### 트리거 사용 시 주의사항

1. **무한 루프(mutually recursive triggers)**: 트리거 A가 B를 유발하고 B가 A를 유발
2. **외래 키 대체 수단**: 트리거로 참조 무결성을 구현하는 것은 복잡도만 높임 → 선언적 제약 선호
3. **실체화 뷰 유지**: 트리거로 집계 테이블을 갱신하는 방식은 일반적
4. **비즈니스 로직**: 애플리케이션 레이어가 더 적합한 경우가 많음

---

## 5.4 재귀 쿼리 (Recursive Queries)

SQL:1999에서 `WITH RECURSIVE`로 도입. 폐쇄(closure) 연산을 표현한다.

### 기본 구조

```sql
WITH RECURSIVE cte_name(columns) AS (
    -- Base case (비재귀 초기 값)
    SELECT ...
    UNION ALL
    -- Recursive step (자기 참조)
    SELECT ... FROM cte_name JOIN ...
)
SELECT * FROM cte_name;
```

### 예시: 과목 선수과목 전이적 폐쇄 (Transitive Closure)

```sql
-- prereq(course_id, prereq_id): 직접 선수과목 관계
WITH RECURSIVE rec_prereq(course_id, prereq_id) AS (
    -- Base: 직접 선수과목
    SELECT course_id, prereq_id FROM prereq
    UNION
    -- Recursive: 간접 선수과목
    SELECT rec_prereq.course_id, prereq.prereq_id
    FROM rec_prereq
    JOIN prereq ON rec_prereq.prereq_id = prereq.course_id
)
SELECT * FROM rec_prereq;
```

```
prereq 테이블:
CS-301 → CS-201
CS-201 → CS-101

rec_prereq 결과:
CS-301 → CS-201  (직접)
CS-201 → CS-101  (직접)
CS-301 → CS-101  (간접, 재귀 1회)
```

### 고정점 의미론 (Fixed-Point Semantics)

재귀 CTE는 다음 과정으로 평가된다:

```
iteration 0: result = base_case
iteration 1: result = result ∪ recursive_step(result)
iteration 2: result = result ∪ recursive_step(result)
...
until result does not change (fixed point)
```

### 재귀 제약사항

SQL 표준의 재귀는 **단조 재귀(monotonic recursion)** 만 허용:
- 재귀 참조가 `UNION ALL`/`UNION`의 오른쪽에 있어야 함
- 재귀 뷰에 대한 집계 불가
- `NOT EXISTS`(부정)로 재귀 뷰 참조 불가
- `EXCEPT`의 오른쪽에 재귀 참조 불가

---

## 5.5 고급 집계 기능 (Advanced Aggregation Features)

### 순위 함수 (Ranking Functions)

```sql
SELECT ID, name, salary,
       RANK() OVER (ORDER BY salary DESC) AS salary_rank
FROM instructor;
```

| 함수 | 동점 처리 | 예) 100, 100, 80 |
|------|-----------|-----------------|
| `RANK()` | 동점은 같은 순위, 다음 순위 건너뜀 | 1, 1, 3 |
| `DENSE_RANK()` | 동점은 같은 순위, 건너뜀 없음 | 1, 1, 2 |
| `ROW_NUMBER()` | 동점 없이 임의 고유 순위 | 1, 2, 3 |
| `PERCENT_RANK()` | (rank - 1) / (rows - 1) | 0.0, 0.0, 1.0 |
| `CUME_DIST()` | ≤ 현재 값인 행의 비율 | 0.67, 0.67, 1.0 |
| `NTILE(n)` | n개 버킷으로 균등 분할 | - |

```sql
-- 학과별 급여 순위
SELECT dept_name, ID, salary,
       RANK() OVER (PARTITION BY dept_name ORDER BY salary DESC) AS rank_in_dept
FROM instructor;
```

`PARTITION BY`는 집계의 `GROUP BY`에 해당. 각 파티션 내에서 독립적으로 순위 부여.

```sql
-- 상위 3명만 (각 학과별)
SELECT * FROM (
    SELECT dept_name, name, salary,
           RANK() OVER (PARTITION BY dept_name ORDER BY salary DESC) AS r
    FROM instructor
) WHERE r <= 3;
```

### NULL 순서 지정

```sql
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC NULLS LAST) AS rank
FROM instructor;
-- NULLS FIRST: NULL을 가장 앞 순위로
-- NULLS LAST:  NULL을 가장 뒤 순위로 (기본값은 구현체 의존)
```

### 윈도우 함수 (Window Functions)

```sql
SELECT year, AVG(num_credits)
       OVER (ORDER BY year ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) AS avg_total_credits
FROM (SELECT year, SUM(credits) AS num_credits FROM takes GROUP BY year);
```

윈도우 프레임(window frame) 지정:

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- 처음부터 현재 행까지
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING           -- 이전 1행, 현재, 다음 1행
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING   -- 현재부터 끝까지
RANGE BETWEEN INTERVAL '1' YEAR PRECEDING AND CURRENT ROW  -- 날짜 범위
```

`ROWS`: 물리적 행 기준 / `RANGE`: 값 기준 (동일 값은 같은 프레임으로 처리)

```sql
-- 누적 합계 (Running Total)
SELECT date, amount,
       SUM(amount) OVER (ORDER BY date
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM transactions;

-- 이동 평균 (Moving Average)
SELECT date, revenue,
       AVG(revenue) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM daily_revenue;
```

---

## 5.6 OLAP (Online Analytical Processing)

### ROLLUP

```sql
SELECT item_name, color, SUM(quantity) AS total_qty
FROM sales
GROUP BY ROLLUP(item_name, color);
```

`ROLLUP(a, b, c)`는 다음 그룹화 집합을 생성:
```
(a, b, c)
(a, b)
(a)
()          ← 전체 합계
```

```
item_name  color   total_qty
Dark       Clothes 20
Dark       Pants   5
Dark       NULL    25    ← item_name별 소계
Pastel     Clothes 10
Pastel     NULL    10
NULL       NULL    35    ← 전체 합계
```

NULL은 집계된 차원을 나타냄. `GROUPING()` 함수로 실제 NULL과 집계 NULL 구분:

```sql
SELECT
    CASE GROUPING(item_name) WHEN 1 THEN 'All Items' ELSE item_name END AS item_name,
    SUM(quantity)
FROM sales
GROUP BY ROLLUP(item_name, color);
```

### CUBE

```sql
SELECT item_name, color, clothes_size, SUM(quantity)
FROM sales
GROUP BY CUBE(item_name, color, clothes_size);
```

`CUBE(a, b, c)`는 2^3 = 8가지 모든 조합을 생성:
```
(a, b, c), (a, b), (a, c), (b, c), (a), (b), (c), ()
```

### GROUPING SETS

```sql
-- 원하는 집계 조합만 명시
SELECT item_name, color, clothes_size, SUM(quantity)
FROM sales
GROUP BY GROUPING SETS (
    (item_name, color),
    (item_name, clothes_size),
    ()
);
```

`ROLLUP`과 `CUBE`의 일반화. 불필요한 조합 없이 원하는 것만 선택.

### PIVOT (교차표, Cross-Tabulation)

```sql
-- 표준 SQL PIVOT (일부 DB만 지원)
SELECT *
FROM (SELECT item_name, color, quantity FROM sales)
PIVOT (
    SUM(quantity)
    FOR color IN ('dark', 'pastel', 'white')
);

-- 결과:
-- item_name | dark | pastel | white
-- Clothes   |  20  |   10   |  5
-- Pants     |   5  |  NULL  |  3
```

PIVOT을 지원하지 않는 DB에서는 `CASE WHEN` + 집계로 구현:

```sql
SELECT item_name,
       SUM(CASE WHEN color = 'dark'   THEN quantity ELSE 0 END) AS dark,
       SUM(CASE WHEN color = 'pastel' THEN quantity ELSE 0 END) AS pastel,
       SUM(CASE WHEN color = 'white'  THEN quantity ELSE 0 END) AS white
FROM sales
GROUP BY item_name;
```

---

## 핵심 정리

### JDBC 계층 구조

```
Java Application
      ↓
  JDBC API (java.sql.*)
      ↓
  JDBC Driver (벤더 제공)
      ↓
  DBMS (Oracle, PostgreSQL, MySQL, ...)
```

### 서버 측 코드 선택 기준

```
SQL 함수(Function) → 값 또는 테이블 반환, SELECT 절에서 사용 가능
SQL 프로시저(Procedure) → 부작용(side-effect) 있는 로직, CALL로 호출
트리거(Trigger) → 이벤트 기반 자동 실행, 직접 호출 불가
```

### 윈도우 함수 구조

```sql
aggregate_func(col) OVER (
    [PARTITION BY partition_cols]
    [ORDER BY sort_cols]
    [ROWS | RANGE BETWEEN frame_start AND frame_end]
)
```

윈도우 함수는 그룹을 만들지 않고(행 수 유지) 각 행에 대해 집계 값을 계산한다는 점에서 `GROUP BY`와 근본적으로 다르다.

### ROLLUP / CUBE / GROUPING SETS 비교

```
GROUP BY a, b        → (a, b) 하나
ROLLUP(a, b)         → (a, b), (a), ()  → n+1 레벨
CUBE(a, b)           → (a, b), (a), (b), ()  → 2^n 조합
GROUPING SETS(...)   → 명시된 조합만
```

### 재귀 쿼리 주의점

- SQL 재귀는 **UNION/UNION ALL** 기반의 단조 증가만 허용
- 집계, NOT EXISTS, EXCEPT는 재귀 참조에 사용 불가
- 무한 루프 방지: 방문 여부 추적이나 깊이 제한 로직 필요
- PostgreSQL의 경우 `CYCLE` 절로 사이클 감지 가능 (SQL:2003)
