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
| 12 | Physical Storage Systems | 저장 계층(캐시→RAM→플래시→HDD→테이프), 자기 디스크(플래터/트랙/섹터/Seek/Rotational Latency/MTTF), SSD/NAND(FTL/Wear Leveling/웨어레벨링), RAID(0/1/5/6·패리티·핫스왑·스크러빙), 디스크 블록 접근 최적화(엘리베이터 알고리즘·NVRAM 버퍼·로그 디스크) | [chapter12_PhysicalStorageSystems.md](./docs/chapter12_PhysicalStorageSystems.md) |
| 13 | Data Storage Structures | 고정/가변 길이 레코드(슬롯페이지·Null Bitmap·자유 리스트), 파일 조직(힙·순차·멀티테이블 클러스터링·파티셔닝), 데이터 사전(메타데이터), 버퍼 관리자(Pin/Unpin·LRU/MRU/Toss-Immediate·강제 출력), 컬럼 지향 저장(ORC/Parquet·벡터처리), 인메모리 DB | [chapter13_DataStorageStructures.md](./docs/chapter13_DataStorageStructures.md) |
| 14 | Indexing | 밀집/희소 인덱스, 클러스터링/보조 인덱스, 다단계 인덱스, B⁺-Tree(구조·find/findRange·삽입분할·삭제합병), B⁺-Tree 파일 조직, 벌크 로딩, 해시 인덱스, 복합 검색 키, 커버링 인덱스, LSM Tree, Buffer Tree, 비트맵 인덱스, R-Tree(공간), 시간 인덱스 | [chapter14_Indexing.md](./docs/chapter14_Indexing.md) |
| 15 | Query Processing | 평가 원시·실행 계획, 비용 모델(tT·tS), 셀렉션 알고리즘(A1~A10·비트맵 인덱스 스캔), 외부 정렬-병합(런생성·N-way병합), 중첩루프/블록중첩루프/인덱스중첩루프/병합/해시 조인, 하이브리드 해시 조인, 집합연산·집계·외부조인, 구체화·파이프라인 평가, 인메모리 캐시인식·쿼리컴파일 | [chapter15_QueryProcessing.md](./docs/chapter15_QueryProcessing.md) |
| 16 | Query Optimization | 동치 변환 규칙(16가지), 선택/투영 조기 처리, 조인 순서·결합법칙, 크기 추정(V(A,r)·히스토그램), 조인 크기 추정, 동적 프로그래밍 조인 순서 최적화(O(3ⁿ)), 왼쪽-깊은 트리, 탈상관화(세미조인·안티세미조인), 점진적 실체화 뷰 유지, Top-K·Join Minimization·Halloween Problem·다중쿼리 최적화 | [chapter16_QueryOptimization.md](./docs/chapter16_QueryOptimization.md) |
| 17 | Transactions | ACID(원자성·일관성·고립성·영속성), 트랜잭션 상태 다이어그램(Active→Partially committed→Committed/Aborted), 스케줄·직렬 스케줄, 충돌(Conflict)·충돌 직렬가능성, 선행 그래프(사이클=비직렬), 회복 가능 스케줄, 연쇄 없는 스케줄, SQL 격리 수준(Read Uncommitted→Serializable), 스냅샷 격리, 팬텀 현상 | [chapter17_Transactions.md](./docs/chapter17_Transactions.md) |
| 18 | Concurrency Control | S/X 잠금·호환성, 잠금 관리자, 2PL(성장/수축·잠금 포인트)·Strict 2PL·Rigorous 2PL, 트리 프로토콜, 교착 상태(Wait-Die·Wound-Wait·대기 그래프), 다중 단위 잠금(IS/IX/SIX), 인덱스 잠금·팬텀 방지, 타임스탬프 순서화·Thomas Write Rule, 낙관적 검증, MVCC·다중버전 2PL, 스냅샷 격리(Write Skew·SSI), Crabbing Protocol·Next-Key Locking·래치-프리 CAS | [chapter18_ConcurrencyControl.md](./docs/chapter18_ConcurrencyControl.md) |
| 19 | Recovery System | 장애 분류(트랜잭션/시스템충돌/디스크), 안정 저장(2중복제), WAL 규칙, 로그 레코드(start/commit/abort/업데이트), redo/undo, 체크포인트·퍼지 체크포인트, No-Force+Steal 표준 정책, 히스토리 반복 2-Phase 복구, 그룹 커밋, 원격 백업(One/Two/Two-very-safe), 논리적 undo·보상 로그, ARIES(LSN·PageLSN·DirtyPageTable·분석/redo/undo 3패스) | [chapter19_RecoverySystem.md](./docs/chapter19_RecoverySystem.md) |
| 20 | Database-System Architectures | 중앙집중식·서버 아키텍처(CAS/TAS 원자 명령), 스피드업·스케일업·Amdahl 법칙, 상호연결(버스/링/메시/트리형), 공유메모리(NUMA·캐시일관성·MESI), 공유디스크(SAN), 공유없음, 계층형, 분산 DB, 클라우드 IaaS·PaaS·SaaS·컨테이너·마이크로서비스 | [chapter20_DatabaseSystemArchitectures.md](./docs/chapter20_DatabaseSystemArchitectures.md) |
| 21 | Parallel and Distributed Storage | 라운드 로빈·해시·범위 파티셔닝, 스큐(가상 노드·일관성 해싱·태블릿 동적 분할), 복제(위치전략·2PC/영속메시징/합의), 과반수·쿼럼 합의, 로컬/글로벌 인덱스, 분산 파일 시스템(HDFS), 병렬 KV 스토어(HBase·Cassandra) | [chapter21_ParallelAndDistributedStorage.md](./docs/chapter21_ParallelAndDistributedStorage.md) |
| 22 | Parallel and Distributed Query Processing | 인트라/인터연산 병렬성, 파티셔닝/브로드캐스팅/대칭 단편 복제 조인, 부분 집계, MapReduce, 교환 연산자 모델(Exchange Operator), 파이프라인(Push/Pull), 공유 메모리 최적화, 반세미조인·분산 쿼리 최적화 | [chapter22_ParallelAndDistributedQueryProcessing.md](./docs/chapter22_ParallelAndDistributedQueryProcessing.md) |
| 23 | Parallel and Distributed Transaction Processing | 분산 트랜잭션(TC/TM 구조), 2PC(준비+결정·블로킹·장애처리), 영속 메시징(정확히 1회 전달), 분산 잠금(단일/분산 관리자), 복제 동시성(기본복제본·과반수·쿼럼·비동기 복제), 최종 일관성, 코디네이터 선출, Paxos·Raft·상태 머신 복제 | [chapter23_ParallelAndDistributedTransactionProcessing.md](./docs/chapter23_ParallelAndDistributedTransactionProcessing.md) |

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
