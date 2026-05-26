# Database System Concepts (7th Edition)

**저자**: Abraham Silberschatz, Henry F. Korth, S. Sudarshan  
**출판**: McGraw-Hill Education, 2020

데이터베이스 시스템의 표준 교재. 관계형 모델, SQL, 데이터베이스 설계, 트랜잭션 관리, 저장 구조, 쿼리 최적화, 병렬/분산 데이터베이스까지 폭넓게 다룬다.

---

## 챕터 목록

| 챕터 | 제목 | 핵심 내용 | 링크 |
|:----:|------|----------|------|
| 1 | Introduction | DBMS 개념, 파일 처리 시스템의 한계, 데이터 추상화(3계층), DDL/DML, DB 엔진 구조, 트랜잭션 기초, 아키텍처 | [chapter01_Introduction.md](./docs/chapter01_Introduction.md) |
| 2 | Introduction to the Relational Model | 릴레이션/튜플/속성, 스키마 vs 인스턴스, 수퍼키·후보키·기본키·외래키, 스키마 다이어그램, 관계 대수(σ Π × ⋈ ∪ − ∩ ← ρ) | [chapter02_IntroductionToTheRelationalModel.md](./docs/chapter02_IntroductionToTheRelationalModel.md) |
| 3 | Introduction to SQL | DDL/DML, 기본 타입, SELECT-FROM-WHERE, DISTINCT, AS, LIKE, ORDER BY, 집합 연산(UNION/INTERSECT/EXCEPT), NULL·3값 논리, 집계 함수, GROUP BY/HAVING, 중첩 서브쿼리(IN/EXISTS/WITH/LATERAL), DELETE/INSERT/UPDATE | [chapter03_IntroductionToSQL.md](./docs/chapter03_IntroductionToSQL.md) |
| 4 | Intermediate SQL | Natural/USING/ON/Outer Join, 뷰·실체화 뷰·갱신 가능 뷰, 트랜잭션(COMMIT/ROLLBACK), 무결성 제약(NOT NULL/UNIQUE/CHECK/FK/CASCADE), 데이터 타입(DATE/LOB/TYPE/DOMAIN/IDENTITY), 권한(GRANT/REVOKE/ROLE) | [chapter04_IntermediateSQL.md](./docs/chapter04_IntermediateSQL.md) |
| 5 | Advanced SQL | JDBC/ODBC/Embedded SQL/Python DB-API, SQL 함수·테이블 함수·프로시저(PSM), 트리거(AFTER/BEFORE/FOR EACH ROW), 재귀 쿼리(WITH RECURSIVE), 순위 함수(RANK/DENSE_RANK), 윈도우 함수(OVER/ROWS), ROLLUP/CUBE/GROUPING SETS, PIVOT | [chapter05_AdvancedSQL.md](./docs/chapter05_AdvancedSQL.md) |
| 6 | Database Design Using the E-R Model | 설계 단계, 엔티티/관계 집합, 복합·다중값·유도 속성, 카디날리티, 약 엔티티, 특수화/일반화/집합체, E-R→관계형 스키마 변환 | [chapter06_DatabaseDesignUsingTheERModel.md](./docs/chapter06_DatabaseDesignUsingTheERModel.md) |
| 7 | Relational Database Design | 함수 종속성, 무손실 분해, BCNF, 3NF, Armstrong 공리, 속성 클로저, 정준 커버, 의존성 보존, 다중값 종속성, 4NF | [chapter07_RelationalDatabaseDesign.md](./docs/chapter07_RelationalDatabaseDesign.md) |
| 8 | Complex Data Types | 반정형 데이터(JSON/XML/RDF·SPARQL), 객체-관계형 DB(UDT·타입/테이블 상속·참조), ORM(Hibernate·Django), 텍스트 데이터(TF-IDF·PageRank), 공간 데이터(래스터/벡터·공간 쿼리) | [chapter08_ComplexDataTypes.md](./docs/chapter08_ComplexDataTypes.md) |
| 9 | Application Development | URL/HTTP/HTML, 세션·쿠키, 서블릿(Java), JSP/PHP/Django, JavaScript·Ajax·REST 웹서비스, MVC·ORM(Hibernate·Django), 커넥션 풀링·memcached, SQL 인젝션·XSS·CSRF·2FA·SSO, AES·공개키·디지털 서명·HTTPS | [chapter09_ApplicationDevelopment.md](./docs/chapter09_ApplicationDevelopment.md) |
| 10 | Big Data | 3V(Volume/Velocity/Variety), HDFS(NameNode/DataNode), 샤딩(범위/해시), Key-Value Store(MongoDB/HBase/Cassandra), CAP 정리, MapReduce(Hadoop), Spark(RDD/DataSet/지연평가), 스트리밍(윈도우/Kafka/Flink), 그래프 DB(Neo4j/Cypher/BSP) | [chapter10_BigData.md](./docs/chapter10_BigData.md) |
| 11 | Data Analytics | ETL/ELT·데이터 웨어하우스·스타/스노우플레이크 스키마, 컬럼 지향 저장, OLAP(피벗/슬라이스/롤업/드릴다운), CUBE/ROLLUP/GROUPING SETS, 데이터 마이닝(의사결정 트리·베이즈·SVM·딥러닝·회귀·연관규칙·클러스터링·텍스트마이닝) | [chapter11_DataAnalytics.md](./docs/chapter11_DataAnalytics.md) |

---

## 책 구성 (Part 개요)

| Part | 주제 |
|------|------|
| Part 1 | Relational Languages (관계형 모델, SQL 기초~고급) |
| Part 2 | Database Design (E-R 모델, 정규화) |
| Part 3 | Application Design (앱 개발, 데이터 마이닝) |
| Part 4 | Big Data (반구조적 데이터, 빅데이터) |
| Part 5 | Storage Management (저장 구조, 인덱싱) |
| Part 6 | Query Processing (쿼리 처리, 최적화) |
| Part 7 | Transaction Management (트랜잭션, 동시성 제어, 복구) |
| Part 8 | Parallel and Distributed Databases |

---

## 핵심 개념 요약

- **DBMS의 존재 이유**: 파일 처리 방식의 중복·불일치·원자성·동시성·보안 문제 해결
- **3계층 추상화**: 물리(Physical) → 논리(Logical) → 뷰(View)
- **트랜잭션**: Atomicity, Consistency, Durability (ACID)
- **DB 엔진**: 저장 관리자 + 쿼리 프로세서 + 트랜잭션 관리자
- **현대 아키텍처**: 3-tier (클라이언트 - 앱 서버 - DB 서버)
