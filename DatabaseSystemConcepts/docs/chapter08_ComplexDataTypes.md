# Chapter 8: Complex Data Types

## 개요

관계형 모델의 핵심 제약인 "원자 값(atomic value)" 요건은 많은 현대 애플리케이션에서 한계가 된다. 이 챕터는 반정형 데이터(Semi-structured Data), 객체 지향(Object Orientation), 텍스트 데이터(Textual Data), 공간 데이터(Spatial Data) 등 비원자적 복합 데이터 타입을 다룬다.

---

## 8.1 반정형 데이터 (Semi-structured Data)

### 8.1.1 반정형 데이터 모델 개요

**왜 필요한가?**
- 관계형 모델은 고정된 스키마로 빠르게 변화하는 웹 애플리케이션에 부적합
- 백엔드와 클라이언트(모바일/브라우저) 간 복잡한 데이터 교환에 구조화된 표현이 필요

반정형 데이터 지원 방식:

| 방식 | 설명 |
|------|------|
| 유연 스키마(Flexible Schema) | 각 튜플이 서로 다른 속성 집합을 가질 수 있음 (Wide Column) |
| 희소 컬럼(Sparse Column) | 고정된 대형 컬럼 집합에서 필요한 것만 사용, 나머지는 NULL |
| 다중값 타입(Multivalued) | 집합, 배열, 키-값 맵을 속성 값으로 허용 |
| 중첩 타입(Nested) | 복합 속성(E-R 모델의 composite attribute)을 직접 표현 |

### 8.1.2 JSON (JavaScript Object Notation)

```json
{
  "ID": "22222",
  "name": {
    "firstname": "Albert",
    "lastname": "Einstein"
  },
  "deptname": "Physics",
  "children": [
    {"firstname": "Hans", "lastname": "Einstein"},
    {"firstname": "Eduard", "lastname": "Einstein"}
  ]
}
```

**JSON 타입**: 정수, 실수, 문자열, 배열(`[]`), 객체(`{}`)

**관계형 DB의 JSON 지원:**

```sql
-- PostgreSQL: JSON 타입 저장
CREATE TABLE instructor_json (ID varchar(5), data JSON);

-- JSON 객체 생성
SELECT json_build_object('ID', ID, 'name', name) FROM instructor;

-- JSON 값 추출 (PostgreSQL: -> 연산자)
SELECT data->'name' FROM instructor_json WHERE ID = '22222';

-- Oracle: JSON_VALUE(value, path)
-- SQL Server: FOR JSON AUTO 절
```

**장단점:**

```
장점: 유연한 구조, 자기 서술적(self-documenting), 라이브러리 풍부
단점: 관계형에 비해 저장 공간 크고 파싱 CPU 비용 높음
       → BSON(Binary JSON)으로 압축 저장 많이 사용
```

### 8.1.3 XML (Extensible Markup Language)

```xml
<course>
  <course_id>CS-101</course_id>
  <title>Intro. to Computer Science</title>
  <dept_name>Comp. Sci.</dept_name>
  <credits>4</credits>
</course>
```

계층적 구조 표현 가능 → 구매 주문서, 청구서 같은 복잡한 비즈니스 객체에 적합.

**SQL의 XML 지원:**
- `XML` 데이터 타입으로 저장
- XPath 경로 표현식으로 값 추출
- `XMLAGG` 집계 함수로 여러 행을 하나의 XML 문서로 결합
- XQuery 언어(채택률은 SQL보다 낮음)

### 8.1.4 RDF와 지식 그래프 (RDF and Knowledge Graphs)

#### 트리플 표현 (Triple Representation)

RDF(Resource Description Framework)는 데이터를 **트리플(triple)** 로 표현:

```
(주어, 술어, 목적어)
(subject, predicate, object)

형식 1: (ID, attribute-name, value)
형식 2: (ID1, relationship-name, ID2)
```

예시 (대학 DB의 일부):
```
10101   instance-of   instructor
10101   name          "Srinivasan"
10101   salary        "65000"
00128   instance-of   student
CS-101  title         "Intro. to Computer Science"
10101   teaches       sec1
```

#### 그래프 표현

```
       Srinivasan      65000
          name↑        salary↑
         10101 ──teaches──> sec1
inst_dept↓               sec_course↓
       comp_sci          CS-101
course_dept↑             title↓
       CS-101     "Intro. to Computer Science"
```

엔티티/속성값 = 노드, 관계/속성명 = 엣지 → **지식 그래프(Knowledge Graph)**

#### SPARQL

```sparql
-- "Intro. to Computer Science" 수강 학생 이름 조회
SELECT ?name
WHERE {
  ?cid title "Intro. to Computer Science" .
  ?sid course ?cid .
  ?id takes ?sid .
  ?id name ?name .
}
```

- 트리플 패턴에서 `?변수`는 임의 값과 매칭
- 공유 변수가 조인 조건 역할

#### N항 관계 표현: Reification

RDF는 이진 관계만 지원 → N항 관계는 **재화(reification)** 로 처리:

```
인위적 엔티티 e1을 생성하여 n항 관계의 각 참여자와 이진 관계로 연결

예) Obama가 2008~2016년 미국 대통령:
e1 ──person──> Obama
e1 ──country──> USA
e1 ──president-from──> 2008
e1 ──president-till──> 2016
```

---

## 8.2 객체 지향 (Object Orientation)

### 8.2.1 객체-관계형 데이터베이스 시스템

#### 사용자 정의 타입 (User-Defined Types, UDT)

```sql
-- Oracle 스타일
CREATE TYPE Person (
  ID      varchar(20) PRIMARY KEY,
  name    varchar(20),
  address varchar(20)
) REF FROM(ID);

CREATE TABLE people OF Person;

-- 배열 타입 (PostgreSQL)
CREATE TABLE users (
  ID       varchar(20),
  interests integer[]    -- 정수 배열
);

-- 테이블 타입 (SQL Server)
CREATE TYPE interest AS TABLE (
  topic              varchar(20),
  degree_of_interest int
);
```

#### 타입 상속 (Type Inheritance)

```sql
CREATE TYPE Student UNDER Person (degree varchar(20));
CREATE TYPE Teacher UNDER Person (salary integer);
-- Student, Teacher는 Person의 속성(ID, name, address)을 상속
```

#### 테이블 상속 (Table Inheritance, PostgreSQL)

```sql
CREATE TABLE students (degree varchar(20)) INHERITS people;
CREATE TABLE teachers (salary integer)     INHERITS people;

-- people 조회 시 students, teachers의 튜플도 암묵적 포함
SELECT * FROM people;           -- 모든 사람
SELECT * FROM ONLY people;      -- 직접 삽입된 튜플만
```

#### 참조 타입 (Reference Types)

```sql
CREATE TYPE Department (
  dept_name varchar(20),
  head      REF(Person) SCOPE people
);
-- head 속성으로 people 테이블의 특정 튜플 직접 참조

-- 역참조 (dereference)
SELECT head->name, head->address FROM departments;
```

참조는 외래 키를 숨겨 조인 없이 관련 튜플에 접근하게 해준다.

### 8.2.2 객체-관계형 매핑 (ORM, Object-Relational Mapping)

ORM은 DB 릴레이션과 프로그래밍 언어 객체 간 변환을 자동화.

```
장점: 프로그래머가 객체 모델로 작업, 이식성(DB 교체 용이), 메모리 캐시 성능
단점: 대량 업데이트/복잡한 쿼리 성능 저하 가능
```

**대표 ORM 시스템:**

| ORM | 언어 | 특징 |
|-----|------|------|
| Hibernate | Java | JPA 구현체, HQL 쿼리 언어 |
| Django ORM | Python | 마이그레이션 도구 포함 |
| SQLAlchemy | Python | 유연한 매핑, Core/ORM 두 레이어 |

```java
// Hibernate 예시
@Entity public class Student {
    @Id String ID;
    String name;
    String department;
    int tot_cred;
}

// 객체 저장 → INSERT INTO student 자동 생성
Session session = getSessionFactory().openSession();
Transaction txn = session.beginTransaction();
Student stud = new Student("12328", "John Smith", "Comp. Sci.", 0);
session.save(stud);
txn.commit();
session.close();
```

```python
# Django ORM 예시
class Student(models.Model):
    id       = models.CharField(primary_key=True, max_length=5)
    name     = models.CharField(max_length=20)
    dept_name = models.CharField(max_length=20)
    tot_cred = models.DecimalField(max_digits=3, decimal_places=0)

class Instructor(models.Model):
    id       = models.CharField(primary_key=True, max_length=5)
    name     = models.CharField(max_length=20)
    advisees = models.ManyToManyField(Student, related_name="advisors")

# 쿼리: 이름이 Zhang인 학생
students = Student.objects.filter(name="Zhang")
for s in students:
    print(s.advisors.all())  # 지도교수 목록
```

---

## 8.3 텍스트 데이터 (Textual Data)

### 키워드 쿼리

정보 검색(Information Retrieval)은 비정형 텍스트 데이터를 질의. 문서(document)가 기본 단위.

**키워드 쿼리**: 지정한 키워드를 모두 포함하는 문서 검색 → 관련성(relevance) 순으로 정렬

### TF-IDF 관련성 랭킹

**TF (Term Frequency)**: 문서 d에서 단어 t의 관련성

```
TF(d, t) = log(1 + n(d,t) / n(d))

n(d,t): 문서 d에서 t의 출현 횟수
n(d): 문서 d의 총 단어 수
```

**IDF (Inverse Document Frequency)**: 단어 t의 희소성 (흔한 단어는 가중치 낮춤)

```
IDF(t) = 1 / n(t)

n(t): t를 포함하는 문서 수
```

**관련성 점수:**

```
r(d, Q) = Σ_{t ∈ Q} TF(d, t) × IDF(t)
```

- **불용어(stop words)**: "and", "or", "a" 같은 고빈도 단어는 색인에서 제외
- **근접도(proximity)**: 쿼리 단어가 문서 내 가까이 있을수록 관련성 높음

### PageRank

하이퍼링크 기반 페이지 중요도 측정. **많이 링크받은 페이지 = 더 중요**.

```
P[j] = δ/N + (1 - δ) × Σ_i (T[i,j] × P[i])

δ: 0.15 (임의 점프 확률)
T[i,j]: 페이지 i에서 j로 링크 팔로우 확률 (= 1/Ni)
```

반복 계산으로 수렴값 산출. PageRank는 쿼리 독립적 정적 지표로, TF-IDF와 결합하여 최종 순위 결정.

### 검색 효과 측정

```
정밀도(Precision@K) = 상위 K개 결과 중 관련 문서 비율
재현율(Recall@K)    = 전체 관련 문서 중 상위 K개에 포함된 비율
```

---

## 8.4 공간 데이터 (Spatial Data)

### 공간 데이터 유형

| 유형 | 설명 | 예시 |
|------|------|------|
| 지리 데이터(Geographic) | 위도/경도 기반 원형 지구 좌표계 | 도로 지도, 위성 이미지 |
| 기하 데이터(Geometric) | 2D/3D 유클리드 공간, X/Y/Z 좌표 | 건물 설계, CAD |

**지원 DB**: Oracle Spatial and Graph, PostGIS(PostgreSQL 확장), SQL Server, IBM DB2 Spatial

### 기하 정보 표현

```
점(Point):      (x, y)
선분(Line Segment): {(x1,y1), (x2,y2)}
폴리라인(Polyline/LineString): 연결된 선분의 연속
다각형(Polygon): 꼭짓점 목록, 삼각분할(triangulation)로 표현 가능
```

```sql
-- PostGIS / SQL Server (OGC 표준 기반)
-- 선: 3점을 잇는 라인
SELECT ST_GeometryFromText('LINESTRING(1 1, 2 3, 4 4)');

-- 다각형(삼각형)
SELECT ST_GeometryFromText('POLYGON((1 1, 2 3, 4 4, 1 1))');

-- 교집합, 합집합
SELECT ST_Intersection(geom1, geom2);
SELECT ST_Union(geom1, geom2);
```

**공간 네트워크(Spatial Network)**: 도로 지도처럼 공간 위치가 있는 그래프. 경로 탐색에 활용.

### 지리 데이터 표현

**래스터 데이터(Raster Data)**:
- 비트맵/픽셀 맵 형태 (예: 위성 이미지)
- 줌 레벨별 타일(tile)로 저장

**벡터 데이터(Vector Data)**:
- 점, 선분, 폴리라인, 다각형 등 기하 객체
- 도로, 국경, 강을 폴리라인/폴리곤으로 표현
- 래스터보다 정밀하지만 면적 기반 데이터는 부적합

**TIN(Triangulated Irregular Network)**: 지형 고도 정보를 삼각형으로 분할하여 압축 표현.

### 공간 쿼리 (Spatial Queries)

```sql
-- 1. 영역 쿼리 (Region Query): 특정 지역 내 객체 검색
SELECT name FROM shops
WHERE ST_Contains(city_boundary, location);  -- 경계 내 상점

-- 2. 근접도 쿼리 (Nearness Query): 특정 거리 내 객체
SELECT name FROM restaurants
WHERE ST_Distance(location, ST_GeographyFromText('POINT(126.9 37.5)')) < 1000;

-- 3. 최근접 이웃 (Nearest Neighbor)
SELECT name FROM gas_stations
ORDER BY ST_Distance(location, my_location)
LIMIT 1;

-- 4. 공간 조인 (Spatial Join)
-- 강우량 데이터와 인구 밀도 데이터 교차
SELECT r.region, p.density, r.rainfall
FROM rainfall r
JOIN population p ON ST_Intersects(r.geom, p.geom);
```

**PostGIS 주요 술어:**

| 함수 | 의미 |
|------|------|
| `ST_Contains(A, B)` | A가 B를 완전히 포함 |
| `ST_Overlaps(A, B)` | A와 B가 부분적으로 겹침 |
| `ST_Disjoint(A, B)` | A와 B가 겹치지 않음 |
| `ST_Touches(A, B)` | A와 B가 경계에서 접함 |
| `ST_Distance(A, B)` | A와 B 사이의 최소 거리 |

---

## 핵심 정리

### 반정형 데이터 형식 비교

```
        JSON              XML               RDF
구조    계층적 객체        계층적 태그        트리플 그래프
용도    API 데이터 교환    문서/설정 교환     지식 표현
쿼리    JSON Path, SQL     XPath, XQuery     SPARQL
스키마  없음(유연)         DTD/XSD(선택)     없음(유연)
성능    경량, 파싱 빠름    무거움            그래프 탐색
```

### ORM 동작 원리

```
애플리케이션 객체 (in-memory)
        ↕  자동 매핑
관계형 테이블 (DB)

객체 조회 → SELECT 생성 → 결과를 객체로 변환
객체 저장 → INSERT/UPDATE 생성
객체 삭제 → DELETE 생성
```

### 공간 인덱스 필요성

일반 B+ 트리는 1차원 정렬 기반 → 2D/3D 공간 데이터 비효율. 이를 위해:
- **R-트리(R-Tree)**: 최소 경계 직사각형(MBR)으로 계층적 공간 분할
- **쿼드트리(Quadtree)**: 균등 4분할 공간 분할
- **k-d 트리**: k차원 이진 공간 분할

*(자세한 공간 인덱스는 Chapter 14에서 다룸)*
