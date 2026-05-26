# Chapter 11: Data Analytics

## 개요

데이터 애널리틱스(Data Analytics)는 과거 데이터로부터 패턴·상관관계·예측 모델을 추출하여 의사결정에 활용하는 분야다. 이 챕터는 데이터 웨어하우스(ETL/ELT), OLAP(다차원 분석), 데이터 마이닝(분류·회귀·연관 규칙·클러스터링·텍스트 마이닝)을 다룬다.

---

## 11.1 애널리틱스 개요

```
운영 시스템 (OLTP)        분석 시스템 (OLAP/DW)
┌─────────────────┐       ┌──────────────────────┐
│ 트랜잭션 처리    │ ETL   │   데이터 웨어하우스    │
│ 소량 데이터 R/W  │ ───►  │  통합 스키마, 이력 보관 │
│ 높은 동시성      │       │  대용량 분석 쿼리     │
└─────────────────┘       └──────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                      ▼
         OLAP 시스템          통계 분석 (R/SAS)        데이터 마이닝
    (다차원 집계·시각화)      (회귀·가설 검정)     (ML 기반 패턴 발견)
```

- **비즈니스 인텔리전스(BI)**: 데이터 애널리틱스와 유사한 개념 (OLAP + 마이닝)
- **의사결정 지원(Decision Support)**: 리포팅·집계에 집중, ML 미포함

---

## 11.2 데이터 웨어하우징

### 11.2.1 데이터 웨어하우스 아키텍처

```
데이터 소스 1 ─┐
데이터 소스 2 ─┤  ETL/ELT   ┌──────────────────────────┐
데이터 소스 3 ─┤ ─────────► │       데이터 웨어하우스    │ ────► 쿼리/분석 도구
    ...        │            │   (통합 스키마, DBMS)      │
외부 데이터   ─┘            └──────────────────────────┘
```

**ETL vs ELT**

| 단계 | ETL | ELT |
|------|-----|-----|
| Extract | 소스에서 데이터 추출 | 소스에서 데이터 추출 |
| Transform | 웨어하우스 로드 **전** 변환 | 웨어하우스 로드 **후** 변환 |
| Load | 변환된 데이터 적재 | 원시 데이터 먼저 적재 |
| 특징 | 전통적 접근 | Hadoop/Spark 기반 병렬 변환 |

**데이터 클렌징(Data Cleansing)**:
- **퍼지 룩업(Fuzzy Lookup)**: 주소·이름 오타 교정 (우편번호 DB 대조)
- **Merge-Purge(중복 제거)**: 다수 소스에서 수집된 중복 레코드 통합
- **Householding**: 동일 주소 여러 사람을 한 세대로 묶어 단일 발송

**업데이트 전파**: 소스 변경 → 웨어하우스 반영. 실질적으로 **뷰 유지(view maintenance)** 문제와 동일.

### 11.2.2 다차원 데이터와 웨어하우스 스키마

#### 스타 스키마 (Star Schema)

```
           item_info                    store
    ┌──────────────────┐       ┌──────────────────┐
    │ item_id (PK)     │       │ store_id (PK)    │
    │ itemname         │       │ city             │
    │ color            │       │ state            │
    │ size             │       │ country          │
    │ category         │       └────────┬─────────┘
    └────────┬─────────┘                │
             │                          │
    ─────────┴──────── sales ───────────┴──────────
             │   ┌──────────────────────────────┐  │
             └──►│ item_id      (FK)            │◄─┘
                 │ store_id     (FK)            │
                 │ customer_id  (FK)            │◄── customer
                 │ date         (FK)            │◄── date_info
                 │ number       [measure]       │
                 │ price        [measure]       │
                 └──────────────────────────────┘
```

- **팩트 테이블(Fact Table)**: 개별 이벤트(매출 등) 기록. 매우 큼.
- **차원 테이블(Dimension Table)**: 팩트 테이블의 FK가 참조하는 룩업 테이블.
- **측정 속성(Measure Attribute)**: 집계 대상 수치 (number, price)
- **차원 속성(Dimension Attribute)**: 그룹핑 기준 (item_id, store_id 등)

**스노우플레이크 스키마(Snowflake Schema)**: 차원 테이블이 다시 다른 테이블을 참조하는 다단계 구조 (예: item_info → manufacturer).

### 11.2.3 컬럼 지향 저장 (Column-Oriented Storage)

```
행 지향 (Row-Oriented):          열 지향 (Column-Oriented):
┌────┬──────┬──────┐             ┌────────────────┐  ┌──────────────────┐
│ id │ name │price │             │ id 파일        │  │ name 파일        │
├────┼──────┼──────┤             │ 1,2,3,4,5,... │  │ A,B,C,D,E,...    │
│ 1  │  A   │  10  │             └────────────────┘  └──────────────────┘
│ 2  │  B   │  20  │             ┌────────────────┐
│ 3  │  C   │  30  │             │ price 파일     │
└────┴──────┴──────┘             │ 10,20,30,...   │
                                 └────────────────┘
```

**장점**:
1. 분석 쿼리에서 필요한 컬럼만 읽기 (I/O 최소화)
2. 동일 타입 값 연속 저장 → 압축 효율 극대화 (RLE, 딕셔너리 인코딩)

**단점**: 단일 튜플 저장/조회 시 다중 I/O.

웨어하우스 전용 DB: **Teradata**, **Amazon Redshift**, **Sybase IQ**. OLAP 지원 범용 DB: Oracle, SAP HANA, SQL Server, DB2.

### 11.2.4 데이터 레이크 (Data Lake)

공통 스키마 없이 다양한 포맷(정형·비정형)으로 데이터를 저장. ETL 비용 없음, 쿼리 시 더 많은 노력 필요. Apache Hadoop/Spark로 쿼리.

---

## 11.3 온라인 분석 처리 (OLAP)

### 11.3.1 다차원 데이터 집계

**예시 스키마**:
```sql
sales (item_name, color, clothes_size, quantity)
```

**크로스 탭(Cross-Tab / Pivot Table)**:

|  | dark | pastel | white | total |
|--|------|--------|-------|-------|
| skirt | 8 | 35 | 10 | 53 |
| dress | 20 | 10 | 5 | 35 |
| shirt | 14 | 7 | 28 | 49 |
| pants | 20 | 2 | 5 | 27 |
| **total** | **62** | **54** | **48** | **164** |

*clothes_size: all (모든 사이즈 합산)*

**데이터 큐브(Data Cube)**: n차원 일반화.

```
         clothes_size
           /
          / small
         /  medium
        /   large
       /    all
      ┌────────────────────────────────────────►
      │                                     color
      │  item_name × color × clothes_size
      │
      ▼
   item_name
```

n개 차원 속성 → 2ⁿ가지 GROUP BY 조합.

### OLAP 연산

| 연산 | 설명 |
|------|------|
| **Pivoting** | 크로스 탭에서 사용할 차원 변경 |
| **Slicing** | 하나의 차원을 특정 값으로 고정 |
| **Dicing** | 복수 차원을 특정 값으로 고정 |
| **Rollup** | 세밀한 단위 → 거친 단위로 집계 (drill up) |
| **Drill Down** | 거친 단위 → 세밀한 단위로 확대 |

**계층(Hierarchy)**: 차원을 여러 세분화 수준으로 구성.

```
시간 계층:
datetime → hour → date → month → quarter → year

위치 계층:
city → state → country → region
```

### 11.3.2 크로스 탭의 관계형 표현

SQL에서는 `all` 대신 NULL을 사용 (`GROUPING()` 함수로 구별).

```sql
-- CUBE: 2ⁿ 가지 모든 GROUP BY 조합 생성
SELECT item_name, color, clothes_size, SUM(quantity)
FROM sales
GROUP BY CUBE(item_name, color, clothes_size);

-- ROLLUP: 계층적 상위 집계 (n+1 가지)
-- (item_name, color, clothes_size) → (item_name, color) → (item_name) → ()
SELECT item_name, color, clothes_size, SUM(quantity)
FROM sales
GROUP BY ROLLUP(item_name, color, clothes_size);

-- GROUPING SETS: 원하는 조합만 지정
SELECT item_name, color, clothes_size, SUM(quantity)
FROM sales
GROUP BY GROUPING SETS ((color, clothes_size), (clothes_size, item_name));
```

**GROUPING() 함수**로 OLAP NULL과 실제 NULL 구별 후 'all' 치환:

```sql
SELECT
    CASE WHEN GROUPING(item_name) = 1 THEN 'all' ELSE item_name END AS item_name,
    CASE WHEN GROUPING(color) = 1 THEN 'all' ELSE color END AS color,
    'all' AS clothes_size,
    SUM(quantity) AS quantity
FROM sales
GROUP BY CUBE(item_name, color);
```

**PIVOT 절** (Oracle/SQL Server 지원):

```sql
SELECT *
FROM sales
PIVOT (
    SUM(quantity)
    FOR color IN ('dark', 'pastel', 'white')
)
ORDER BY item_name;
```

### 11.3.3 OLAP 구현 방식

| 유형 | 설명 |
|------|------|
| **MOLAP** | 메모리 내 다차원 배열에 데이터 큐브 저장 (초고속) |
| **ROLAP** | 관계형 DB에 저장, SQL로 집계 |
| **HOLAP** | 일부 요약은 메모리, 원본+나머지는 RDB |

**사전 계산(Precomputation)**: 2ⁿ가지 전체 큐브 저장 시 공간 폭발. 실무에서는 자주 사용되는 일부 집계만 저장, 나머지는 저장된 집계로부터 온디맨드 계산.

### 11.3.4 리포팅·시각화 도구

- **리포트 생성기**: SAP Crystal Reports, SQL Server Reporting Services
- **BI 도구**: Tableau, PowerBI, FusionCharts, plotly, Google Charts
- **핵심 기능**: 드릴다운, 필터링, 피봇, 지도/그래프 시각화
- **대시보드**: 조직 KPI를 실시간 차트로 표시

---

## 11.4 데이터 마이닝 (Data Mining)

**정의**: 대용량 데이터베이스에서 유용한 패턴·규칙·모델을 반자동적으로 발견하는 과정. 전통적 ML과의 차이: 디스크 기반 초대용량 데이터 처리에 특화.

**KDD(Knowledge Discovery in Databases)**: 데이터 마이닝이 포함된 전체 지식 발견 프로세스.

### 11.4.1 데이터 마이닝 작업 유형

| 유형 | 목적 | 예시 |
|------|------|------|
| **분류(Classification)** | 클래스 예측 | 신용등급 판단, 스팸 분류 |
| **회귀(Regression)** | 연속값 예측 | 부동산 가격 예측 |
| **연관 규칙(Association Rules)** | 동시 발생 패턴 | 장바구니 분석 |
| **클러스터링(Clustering)** | 자연 그룹 발견 | 고객 세그멘테이션 |
| **텍스트 마이닝(Text Mining)** | 비정형 텍스트 분석 | 감성 분석 |

### 11.4.2 분류 (Classification)

**훈련 데이터(Training Instances)**: 클래스 레이블이 있는 과거 사례.
**테스트 데이터**: 분류 대상 신규 인스턴스 (클래스 미지).

#### 11.4.2.1 의사결정 트리 분류기 (Decision-Tree Classifier)

```
                    degree
           ┌─────────┼─────────┐
         none     bachelors   masters/doctorate
           │           │           │
         income      income      income
        ┌──┴──┐    ┌───┴───┐   ┌───┴────┐
      <50K  ≥50K  <50K  50~100K  <25K  25~75K  >75K
       bad  avg   bad     avg    avg   good   excellent
```

- 루트에서 리프까지 탐색하며 분류
- 각 내부 노드: 속성에 대한 조건/술어
- 각 리프: 클래스 레이블
- **학습**: 훈련 데이터의 엔트로피/지니 불순도를 최소화하는 분할 선택

#### 11.4.2.2 베이즈 분류기 (Bayesian Classifier)

베이즈 정리 기반 확률적 분류:

```
                    p(d|c_j) × p(c_j)
p(c_j | d) = ─────────────────────────
                        p(d)
```

- `p(c_j)`: 클래스 j의 사전 확률 (훈련 데이터 내 비율)
- `p(d|c_j)`: 클래스 j에서 인스턴스 d가 나타날 우도(likelihood)
- `p(d)`: 모든 클래스에 공통 → 무시 가능

**나이브 베이즈(Naive Bayesian)**: 속성들의 조건부 독립 가정:

```
p(d|c_j) ≈ p(d₁|c_j) × p(d₂|c_j) × ⋯ × p(dₙ|c_j)
```

각 속성의 분포를 히스토그램으로 근사하여 계산. 속성 수가 많아도 계산 가능.

#### 11.4.2.3 서포트 벡터 머신 (SVM)

```
클래스 A (×)  │  클래스 B (○)
              │
  × × ×       │       ○ ○ ○
    × ←── 최대 마진 초평면 ──→ ○
  × × ×       │       ○ ○ ○
              │
   |← margin→| | ←margin→|
```

- **최대 마진 초평면(Maximum Margin Hyperplane)**: 두 클래스에서 가장 가까운 점(서포트 벡터)까지 거리를 최대화
- **커널 함수(Kernel Functions)**: 비선형 분리가 필요할 때 입력 공간을 고차원으로 변환
- N진 분류: N개의 이진 분류기 앙상블 (one-vs-rest)

#### 11.4.2.4 신경망 분류기 (Neural Networks)

```
입력층        은닉층 1      은닉층 2      출력층
[x₁] ─────►  [h₁]                      [y₁] class A
[x₂] ─────►  [h₂] ───────► [h₁']  ───► [y₂] class B
[x₃] ─────►  [h₃]          [h₂']       [y₃] class C
```

- 각 뉴런: 입력의 가중합 + 활성화 함수
- **역전파(Backpropagation)**: 오류 역방향 전파로 가중치 업데이트
- **심층 학습(Deep Learning)**: 다수 은닉층 신경망, 대규모 훈련 데이터 필요
- 적용: 이미지 인식, 음성 인식, 자연어 처리

### 11.4.3 회귀 (Regression)

연속 값 예측. **선형 회귀(Linear Regression)**:

```
Y = a₀ + a₁×X₁ + a₂×X₂ + ⋯ + aₙ×Xₙ
```

계수 `aᵢ`는 훈련 데이터로부터 **최소 제곱법(Least Squares)**으로 추정. 일반화: **곡선 피팅(Curve Fitting)** (다항식 또는 임의 함수).

### 11.4.4 연관 규칙 (Association Rules)

**형식**:
```
빵 ⇒ 우유  [support: 50%, confidence: 80%]
```

- **지지도(Support)**: 모집단 중 전건(antecedent)과 후건(consequent)를 **모두** 포함하는 비율
- **신뢰도(Confidence)**: 전건이 참일 때 후건이 참인 비율

```
            |X ∩ Y|                    |X ∩ Y|
support = ─────────    confidence = ────────────
              |D|                       |X|
```

**주의**: `빵 ⇒ 우유`와 `우유 ⇒ 빵`의 신뢰도는 다를 수 있음 (지지도는 동일).

**응용**: 장바구니 분석, 추천 시스템, 매장 레이아웃 최적화, 할인 정책.

#### Apriori 알고리즘 (핵심 아이디어)

```
빈발 항목 집합(Frequent Itemsets) 발견:
1. 최소 지지도(min_support) 이상인 단일 항목 찾기
2. 빈발 단일 항목 조합으로 후보 2-항목 집합 생성
3. 지지도 계산 후 비빈발 제거
4. 크기 증가 반복...

→ 빈발 항목 집합에서 신뢰도 조건 만족하는 규칙 생성
```

### 11.4.5 클러스터링 (Clustering)

레이블 없이 유사한 인스턴스를 자동으로 그룹화.

**목표**: 클러스터 내 거리 최소화, 클러스터 간 거리 최대화.

**k-means 알고리즘**:
```
1. k개 초기 중심점(centroid) 무작위 선정
2. 각 점을 가장 가까운 중심점의 클러스터에 배정
3. 각 클러스터의 새 중심점 계산 (평균)
4. 수렴까지 2~3 반복
```

중심점 = 클러스터 내 모든 점의 각 차원 평균.

**계층적 클러스터링(Hierarchical Clustering)**: 트리 구조로 중첩 클러스터 생성. 생물 분류학에 활용.

**협업 필터링(Collaborative Filtering)**: 영화 추천 예시.
```
1. 사용자들을 과거 시청 선호도로 클러스터링
2. 영화들을 유사성으로 클러스터링
3. 신규 사용자 → 유사 사용자 클러스터 찾기
4. 해당 클러스터에서 인기 있는 영화 클러스터 추천
```

### 11.4.6 텍스트 마이닝 (Text Mining)

비정형 텍스트에 데이터 마이닝 기법 적용.

**감성 분석(Sentiment Analysis)**:
```
리뷰 텍스트 → 긍정/부정/중립 분류
긍정 키워드: excellent, good, awesome, beautiful
부정 키워드: awful, average, worthless, poor quality
```

**정보 추출(Information Extraction)**:
- **개체명 인식(Named Entity Recognition, NER)**: 텍스트에서 사람·장소·조직 등 식별
- **모호성 해소(Disambiguation)**: "Michael Jordan" → 농구선수 vs ML 교수 (문맥 기반)
- **관계 학습**: 엔티티 간 관계 추출 → **지식 그래프(Knowledge Graph)** 구축

**활용**: 고객 지원 분석, 소셜 미디어 모니터링, 검색 엔진 의미론적 이해.

---

## 11.5 핵심 요약

```
데이터 애널리틱스 전체 흐름

소스 데이터 ─► ETL/ELT ─► 데이터 웨어하우스(DW)
                                    │
                   ┌────────────────┼──────────────────┐
                   ▼                ▼                  ▼
               OLAP 분석         통계 분석          데이터 마이닝
            (집계·시각화)       (R/SAS/SPSS)      (ML 기반 패턴)
                   │                                   │
            ┌──────┴──────┐                  ┌─────────┼──────────┐
            │             │              분류  │  회귀   │  연관규칙 │
         크로스탭       데이터큐브             │         │          │
         ROLLUP/CUBE    드릴다운/롤업      클러스터링  텍스트마이닝
         시각화         슬라이싱/다이싱
```

| 개념 | 핵심 포인트 |
|------|------------|
| 데이터 웨어하우스 | 멀티소스 통합, ETL/ELT, 이력 보관, 분석 최적화 스키마 |
| 스타/스노우플레이크 스키마 | 팩트 테이블 + 차원 테이블, FK 참조 |
| 컬럼 지향 저장 | 분석 쿼리 I/O 최소화, 압축 효율, OLAP에 최적 |
| OLAP | 다차원 데이터, 피벗/슬라이스/다이스/롤업/드릴다운 |
| CUBE/ROLLUP | 2ⁿ/n+1 가지 GROUP BY 한번에 처리 |
| 의사결정 트리 | 계층적 조건 분기, 해석 용이 |
| 나이브 베이즈 | 조건부 독립 가정, 확률 기반 |
| SVM | 최대 마진 초평면, 커널 트릭으로 비선형 분리 |
| 딥러닝 | 다층 신경망, 역전파, 대규모 데이터 의존 |
| 연관 규칙 | 지지도·신뢰도 임계값 기반 빈발 패턴 탐색 |
| 클러스터링 | 비지도 그룹화, k-means, 협업 필터링 응용 |
| 텍스트 마이닝 | NER, 감성 분석, 지식 그래프 구축 |
