# Chapter 10: Big Data

## 개요

빅데이터(Big Data)는 전통적인 단일 머신 DBMS가 처리할 수 없는 규모·속도·다양성을 가진 데이터를 다룬다. 이 챕터는 분산 저장(HDFS), 분산 Key-Value 스토어(NoSQL), MapReduce 패러다임, Spark, 스트리밍 처리, 그래프 데이터베이스를 다룬다.

---

## 10.1 빅데이터의 특성 (3V)

| 특성 | 설명 | 예시 |
|------|------|------|
| Volume | 단일 조직의 기존 처리 능력을 초과하는 데이터 규모 | TB~PB 규모의 웹 로그, IoT 센서 |
| Velocity | 데이터 생성·소비 속도 | 실시간 트랜잭션 스트림, 소셜 미디어 피드 |
| Variety | 다양한 형식 (구조화·반구조화·비구조화) | JSON 로그, 이미지, 자연어 텍스트 |

**IoT(Internet of Things)**: 센서·차량·건물 내 임베디드 디바이스가 인터넷에 연결되어 대용량 스트리밍 데이터를 생성한다.

---

## 10.2 빅데이터 저장 시스템

### 10.2.1 분산 파일 시스템 (HDFS)

HDFS(Hadoop Distributed File System)는 대용량 파일을 다수 노드에 걸쳐 저장하면서도 단일 파일 시스템 인터페이스를 제공한다.

```
┌─────────────────────────────────────────────────┐
│                   클라이언트                      │
└──────────────────────┬──────────────────────────┘
                       │ 메타데이터 요청
                       ▼
              ┌─────────────────┐
              │   NameNode      │  ← 단일 마스터
              │ (파일→블록 매핑) │    (메타데이터 전담)
              └────────┬────────┘
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │DataNode 1│  │DataNode 2│  │DataNode 3│
  │ Block A  │  │ Block A' │  │ Block A''│  ← 3중 복제
  │ Block B  │  │ Block C  │  │ Block B' │
  └──────────┘  └──────────┘  └──────────┘
```

- **블록 크기**: 기본 128MB (설정 가능)
- **3중 복제**: 노드 장애 시에도 데이터 손실 없음
- **쓰기 패턴**: 1회 쓰기 후 여러 번 읽기 최적화 (append는 가능, in-place 수정 미지원)
- **NameNode 단일 장애점**: High Availability 구성 시 Secondary NameNode/Standby NameNode 사용

### 10.2.2 샤딩 (Sharding)

대용량 관계형 데이터를 여러 노드에 파티셔닝하는 기법.

**범위 파티셔닝(Range Partitioning)**:
```sql
-- 예: customer_id 기준 분할
-- Shard 1: customer_id < 1,000,000
-- Shard 2: 1,000,000 <= customer_id < 2,000,000
-- Shard 3: customer_id >= 2,000,000
```

**해시 파티셔닝(Hash Partitioning)**:
```
shard_id = hash(partition_key) mod n
```
- 범위 쿼리에 비효율적이지만 균등 분포 보장

**문제점**: 다중 샤드를 아우르는 조인·트랜잭션이 복잡해짐 (cross-shard join = 분산 조인).

### 10.2.3 Key-Value 스토어 (NoSQL)

전통적 RDBMS 대신 확장성 우선 설계. 인터페이스:

```
put(key, value)   -- 저장
get(key)          -- 조회
delete(key)       -- 삭제
```

| 시스템 | 특징 |
|--------|------|
| **MongoDB** | Document Store, JSON 저장, 풍부한 쿼리 지원 |
| **Google Bigtable** | 행키+열패밀리+타임스탬프 3D 주소, 정렬된 범위 스캔 |
| **HBase** | Bigtable 오픈소스 구현 (Hadoop 위) |
| **Cassandra** | 분산 해시 테이블, eventual consistency 기본 |
| **Amazon DynamoDB** | 관리형 Key-Value, 자동 샤딩 |

**MongoDB 예시**:
```javascript
// 컬렉션에 문서 삽입
db.students.insertOne({
  _id: "12345",
  name: "Kim",
  courses: ["CS101", "DB201"],
  address: { city: "Seoul", zip: "04524" }
})

// 쿼리
db.students.find({ "address.city": "Seoul" })
db.students.findOne({ _id: "12345" })
db.students.remove({ name: "Kim" })
db.students.drop()  // 컬렉션 전체 삭제
```

### 10.2.4 CAP 정리

분산 시스템은 아래 세 속성 중 **최대 두 가지**만 동시에 보장 가능하다.

```
         C (Consistency)
        / \
       /   \
      /     \
     /       \
    P---------A
(Partition   (Availability)
 Tolerance)
```

| 조합 | 설명 | 예시 |
|------|------|------|
| CA | 파티션 없는 환경에서 일관성+가용성 | 단일 노드 RDBMS |
| CP | 파티션 발생 시 가용성 포기 | HBase, Zookeeper |
| AP | 파티션 발생 시 일관성 포기 | Cassandra, DynamoDB |

- **Eventual Consistency**: 모든 업데이트가 최종적으로 모든 복제본에 전파됨을 보장하지만, 중간 시점에는 불일치 허용

---

## 10.3 MapReduce

### 기본 패러다임

```
입력 데이터
    │
    ▼
┌──────────┐     ┌──────────────────────────────┐
│ map(k,v) │ → (key, value) 쌍 다수 출력         │
└──────────┘     └──────────────────────────────┘
                         │
                         ▼ (셔플/정렬: 같은 키 묶음)
                 ┌───────────────┐
                 │ reduce(k,[v]) │ → 최종 결과
                 └───────────────┘
```

### Word Count 예시 (의사코드)

```
map(docid, document):
    for each word w in document:
        emit(w, 1)

reduce(word, [1, 1, 1, ...]):
    count = sum of all values
    emit(word, count)
```

### Hadoop Java 구현

```java
// Mapper
public class WordCountMapper
        extends Mapper<LongWritable, Text, Text, IntWritable> {
    private final IntWritable ONE = new IntWritable(1);
    private Text word = new Text();

    @Override
    public void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {
        StringTokenizer tokens = new StringTokenizer(value.toString());
        while (tokens.hasMoreTokens()) {
            word.set(tokens.nextToken());
            context.write(word, ONE);  // emit(word, 1)
        }
    }
}

// Reducer
public class WordCountReducer
        extends Reducer<Text, IntWritable, Text, IntWritable> {
    @Override
    public void reduce(Text key, Iterable<IntWritable> values, Context context)
            throws IOException, InterruptedException {
        int sum = 0;
        for (IntWritable val : values) {
            sum += val.get();
        }
        context.write(key, new IntWritable(sum));
    }
}

// Driver
public class WordCount {
    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "word count");
        job.setJarByClass(WordCount.class);
        job.setMapperClass(WordCountMapper.class);
        job.setCombinerClass(WordCountReducer.class); // 로컬 사전 집계
        job.setReducerClass(WordCountReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

**Combiner**: 각 Mapper 노드에서 로컬 집계를 먼저 수행하여 네트워크 전송량 감소. Reduce 함수와 동일한 로직 적용 가능 (교환·결합 법칙 성립 시).

### SQL on MapReduce

**Apache Hive**: SQL을 MapReduce/Tez/Spark 작업으로 컴파일. HiveQL은 표준 SQL과 거의 동일.

```sql
-- Hive 예시: 날짜별 페이지뷰 집계
SELECT dt, page, COUNT(*) AS cnt
FROM weblogs
WHERE status = 200
GROUP BY dt, page
ORDER BY dt, cnt DESC;
```

**Apache Pig**: Pig Latin이라는 데이터 흐름 언어로 MapReduce 파이프라인을 표현.

---

## 10.4 MapReduce를 넘어서: Apache Spark

### 10.4.1 RDD (Resilient Distributed Dataset)

Spark의 핵심 추상화. 여러 파티션으로 나뉜 불변(immutable) 분산 컬렉션.

```
┌───────────────────────────────────────┐
│         Spark RDD                     │
│  Partition 1 │ Partition 2 │ ...      │
│  [Node A]    │ [Node B]    │ ...      │
└───────────────────────────────────────┘
```

**특성**:
- **불변성(Immutability)**: 변환 시 새 RDD 생성
- **내결함성(Fault Tolerance)**: 리니지(lineage) 그래프를 통해 손실된 파티션 재계산
- **지연 평가(Lazy Evaluation)**: 액션(action) 호출 전까지 변환(transformation) 미실행

### 10.4.2 Word Count (Spark Java)

```java
SparkConf conf = new SparkConf().setAppName("WordCount");
JavaSparkContext sc = new JavaSparkContext(conf);

JavaRDD<String> lines = sc.textFile("hdfs:///input");

JavaPairRDD<String, Integer> counts = lines
    .flatMap(line -> Arrays.asList(line.split(" ")).iterator())  // transformation
    .mapToPair(word -> new Tuple2<>(word, 1))                    // transformation
    .reduceByKey((a, b) -> a + b);                               // transformation

counts.saveAsTextFile("hdfs:///output");  // action: DAG 실행 트리거
```

### 10.4.3 DAG 기반 지연 평가

```
textFile  →  flatMap  →  mapToPair  →  reduceByKey
                                              │
                              saveAsTextFile() 호출 시점에
                              전체 DAG 실행 (스케줄러가 최적화)
```

**장점**:
- 불필요한 중간 결과 계산 생략
- 파이프라인 내 연속 변환을 단일 태스크로 융합(fusion)
- 전체 DAG를 보고 최적 실행 계획 수립

### 10.4.4 DataSet (구조화 API)

```java
SparkSession spark = SparkSession.builder().appName("example").getOrCreate();

// Parquet 파일 로드
Dataset<Row> df = spark.read().parquet("hdfs:///data/sales.parquet");

// SQL 스타일 조작
Dataset<Row> result = df
    .filter(col("amount").gt(1000))
    .join(itemDf, "item_id")
    .groupBy("category")
    .agg(sum("amount").alias("total"));

result.show();
```

지원 파일 포맷: **Parquet**, **ORC**, **Avro** (컬럼 지향 압축 포맷으로 분석 쿼리에 최적).

---

## 10.5 스트리밍 데이터 (Streaming Data)

### 10.5.1 스트리밍의 활용 사례

| 도메인 | 예시 |
|--------|------|
| 주식/금융 | 실시간 알고리즘 트레이딩, 불법 거래 패턴 탐지 |
| E-commerce | 캠페인 실시간 효과 측정, 재고 급증 감지 |
| IoT/센서 | 장비 이상 알람, 클라우드 기반 중앙 모니터링 |
| 네트워크 | 링크 장애 탐지, 악성코드 전파 패턴 분석 |
| 소셜미디어 | 게시물 라우팅, 부정 감성 알림 |

### 10.5.2 스트림 질의 방법

**데이터-at-rest**: 전통적인 저장 데이터. **스트리밍**: 끝이 없는(unbounded) 데이터 흐름.

1. **연속 쿼리(Continuous Queries)**: 스트림을 관계로 취급하여 SQL 쿼리를 계속 실행. 입력 튜플마다 결과 업데이트 스트림 생성.

2. **스트림 쿼리 언어**: SQL 확장으로 **윈도우 연산** 지원.

#### 윈도우 타입

| 타입 | 설명 | 예시 |
|------|------|------|
| Tumbling Window | 겹치지 않는 고정 크기 윈도우 | 매 시간 단위 집계 |
| Hopping Window | 슬라이딩 간격이 있는 고정 크기 윈도우 | 매 20분마다 1시간 집계 |
| Sliding Window | 각 튜플 중심의 고정 크기 | SQL 표준 지원 |
| Session Window | 활동 간 타임아웃 기반 | 사용자 세션 분석 |

**Azure Stream Analytics 예시 (Tumbling Window)**:
```sql
SELECT itemid,
       System.Timestamp AS window_end,
       SUM(amount) AS total
FROM order TIMESTAMP BY datetime
GROUP BY itemid, TUMBLINGWINDOW(hour, 1)
```

3. **대수 연산자(Algebraic Operators)**: 사용자 정의 연산자를 DAG로 연결. Apache Storm (topology/spout/bolt), Apache Kafka (pub-sub).

4. **복합 이벤트 처리(CEP)**: 패턴 매칭 규칙 기반. Oracle Event Processing, FlinkCEP.

#### 람다 아키텍처 (Lambda Architecture)

```
                  ┌─── 스트리밍 시스템 (저지연 처리)
입력 스트림 ───┤
                  └─── 데이터베이스 (영속 저장 + 배치 쿼리)
```

문제점: 쿼리를 두 번 작성, 스트리밍 시스템에서 저장 데이터 비효율 접근.

### 10.5.3 스트림 대수 연산

- **Apache Storm**: topology 설정 파일로 그래프 정의, spout(소스) → bolt(연산자)
- **Apache Kafka**: 토픽 기반 pub-sub, 보존 기간(retention period) 동안 튜플 유지
- **Apache Spark Streaming**: 스트림을 **이산화 스트림(discretized streams)**으로 분할하여 배치 처리
- **Apache Flink**: 네이티브 스트림 처리, 윈도우 연산으로 집계 출력

---

## 10.6 그래프 데이터베이스

### 10.6.1 관계형으로 표현

```sql
CREATE TABLE node (ID INT, label VARCHAR(100), node_data TEXT);
CREATE TABLE edge (fromID INT, toID INT, label VARCHAR(100), edge_data TEXT);
```

복잡한 스키마에서는 노드/엣지 타입별로 별도 관계 사용.

### 10.6.2 Neo4j와 Cypher 쿼리 언어

**Neo4j**는 그래프 전용 DB로 경로 쿼리(path query)를 간결하게 표현.

```cypher
-- 컴퓨터과학과 교수-학생 어드바이저 관계 조회
MATCH (i:instructor)<-[:advisor]-(s:student)
WHERE i.dept_name = 'Comp. Sci.'
RETURN i.ID AS ID,
       i.name AS name,
       collect(s.name) AS advisees

-- 재귀 선수과목 탐색 (1개 이상 엣지)
MATCH (c1:course)-[:prereq*1..]->(c2:course)
RETURN c1.course_id, c2.course_id
```

`*1..` 표기: 1개 이상의 `prereq` 엣지를 재귀적으로 순회 (SQL WITH RECURSIVE에 해당).

### 10.6.3 병렬 그래프 처리

**문제**: 수백억 노드·수조 엣지 규모 (웹 그래프, 소셜 네트워크).

**방법 1: MapReduce/Spark 기반**
- 그래프를 관계로 표현하여 조인 연산으로 알고리즘 구현
- 반복 알고리즘에서 매 이터레이션마다 전체 그래프 읽기 → 비효율

**방법 2: BSP (Bulk Synchronous Processing)**

```
각 슈퍼스텝(superstep)에서:
1. 각 정점(vertex)이 인접 정점으로부터 메시지 수신
2. 자신의 상태(state) 업데이트
3. 인접 정점으로 메시지 발송 (다음 슈퍼스텝에 수신됨)
4. 더 이상 처리 불필요 시 halt 투표

모든 정점이 halt 투표 + 메시지 없음 → 종료
결과: 각 정점의 최종 상태
```

- **Pregel** (Google): BSP 대중화, 내결함성 구현
- **Apache Giraph**: Pregel 오픈소스 구현
- **Apache Spark GraphX**: Pregel API + 그래프 대수 연산 (map on vertices/edges, aggregation)

---

## 10.7 핵심 요약

```
빅데이터 생태계
│
├── 저장
│   ├── HDFS (대용량 파일)
│   ├── Key-Value Store (MongoDB, HBase, Cassandra)
│   └── 병렬 DB (Google Spanner, CockroachDB)
│
├── 배치 처리
│   ├── Hadoop MapReduce
│   ├── Apache Spark (RDD / DataSet)
│   └── SQL on Hadoop (Hive, Impala)
│
├── 스트림 처리
│   ├── Apache Kafka (pub-sub 라우팅)
│   ├── Apache Flink (네이티브 스트림)
│   └── Spark Streaming (마이크로 배치)
│
└── 그래프
    ├── Neo4j (중앙화, Cypher)
    └── GraphX / Giraph (분산 BSP)
```

| 개념 | 핵심 포인트 |
|------|------------|
| HDFS | NameNode 메타데이터, DataNode 블록 3복제, 1-write-many-read |
| Sharding | 범위/해시 파티셔닝, cross-shard 조인 문제 |
| CAP 정리 | 파티션 내성 전제 시 C vs A 트레이드오프 |
| MapReduce | map → shuffle/sort → reduce, Combiner로 네트워크 최적화 |
| Spark RDD | 불변, 지연 평가, 리니지 기반 내결함성 |
| 스트리밍 | 윈도우(Tumbling/Hopping/Sliding/Session) 기반 유한 집계 |
| BSP | 슈퍼스텝 단위 정점 중심 메시지 패싱 |
