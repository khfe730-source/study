# Google Cloud Platform (GCP) 핵심 정리

---

## 1. GCP 주요 서비스 개요

| 카테고리 | 서비스 | 용도 |
|---------|--------|------|
| 컴퓨팅 | GKE | 관리형 쿠버네티스 |
| 컴퓨팅 | GCE (Compute Engine) | 가상 머신 |
| 컴퓨팅 | Cloud Run | 서버리스 컨테이너 |
| 스토리지 | Cloud Storage (GCS) | 오브젝트 스토리지 |
| DB | Cloud SQL | 관리형 MySQL/PostgreSQL |
| DB | Cloud Spanner | 글로벌 분산 RDBMS |
| DB | Firestore | NoSQL 문서 DB |
| DB | Bigtable | 대용량 NoSQL |
| 캐시 | Memorystore | 관리형 Redis/Memcached |
| 메시지 | Pub/Sub | 메시지 큐 |
| 네트워크 | Cloud Load Balancing | 글로벌 LB |
| 네트워크 | Cloud CDN | 콘텐츠 배포 |
| 보안 | Cloud Armor | DDoS 방어, WAF |
| 모니터링 | Cloud Monitoring | 메트릭, 알람 |
| 로깅 | Cloud Logging | 로그 수집/분석 |

---

## 2. GKE (Google Kubernetes Engine)

```bash
# GKE 클러스터 생성
gcloud container clusters create game-cluster \
  --region asia-northeast3 \
  --num-nodes 3 \
  --machine-type n2-standard-4 \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 10

# kubeconfig 설정
gcloud container clusters get-credentials game-cluster \
  --region asia-northeast3

# Node Pool 추가 (GPU, 고메모리 등 특수 목적)
gcloud container node-pools create battle-pool \
  --cluster game-cluster \
  --region asia-northeast3 \
  --machine-type c2-standard-8 \
  --num-nodes 2
```

**GKE 주요 기능**
- **Autopilot 모드**: 노드 관리 불필요, Pod 단위 과금
- **Standard 모드**: 노드 직접 관리
- **Workload Identity**: GCP 서비스 계정을 K8s 서비스 계정에 매핑 (키 파일 없이 GCP 서비스 접근)
- **GKE Autopilot**: 보안, 업그레이드, 스케일링 자동화

---

## 3. Cloud Storage (GCS)

```bash
# 버킷 생성
gsutil mb -l asia-northeast3 gs://my-game-bucket

# 파일 업로드/다운로드
gsutil cp localfile.txt gs://my-game-bucket/
gsutil cp gs://my-game-bucket/file.txt .

# 버킷 권한 설정
gsutil iam ch allUsers:objectViewer gs://public-bucket

# 서명된 URL (임시 접근)
gsutil signurl -d 1h service-account.json gs://bucket/file
```

**스토리지 클래스**
| 클래스 | 접근 빈도 | 최소 보관 |
|--------|---------|---------|
| Standard | 자주 | 없음 |
| Nearline | 월 1회 미만 | 30일 |
| Coldline | 분기 1회 미만 | 90일 |
| Archive | 연 1회 미만 | 365일 |

---

## 4. Cloud SQL

```bash
# 인스턴스 생성
gcloud sql instances create game-db \
  --database-version=MYSQL_8_0 \
  --region=asia-northeast3 \
  --tier=db-n1-standard-2 \
  --storage-type=SSD \
  --storage-size=100GB \
  --backup-start-time=03:00

# DB 생성
gcloud sql databases create gamedb --instance=game-db

# Cloud SQL Proxy (로컬에서 접속)
cloud_sql_proxy -instances=PROJECT:REGION:INSTANCE=tcp:3306
```

**HA 구성**: Primary + Standby 자동 페일오버. Read Replica로 읽기 부하 분산.

---

## 5. Memorystore (관리형 Redis)

```bash
# Redis 인스턴스 생성
gcloud redis instances create game-cache \
  --size=5 \
  --region=asia-northeast3 \
  --tier=STANDARD  # BASIC(단일) vs STANDARD(HA)
```

게임 서버에서의 활용:
- 세션 정보, 접속 상태 관리
- 랭킹 (Redis Sorted Set)
- 분산 락
- 실시간 카운터, 쿨다운 관리
- 채팅 채널 (Redis Pub/Sub)

---

## 6. Cloud Pub/Sub

```bash
# 토픽 생성
gcloud pubsub topics create game-events

# 구독 생성
gcloud pubsub subscriptions create game-events-sub \
  --topic=game-events \
  --ack-deadline=60
```

**게임 서버 활용 패턴**
- 서버 간 이벤트 전달 (전투 결과, 아이템 획득)
- 비동기 처리 (로그 수집, 통계 집계)
- 마이크로서비스 간 디커플링

---

## 7. Cloud Load Balancing

```
인터넷
  ↓
[Global HTTP(S) Load Balancer]  ← SSL Termination, Cloud Armor
  ↓
[Cloud CDN]  ← 정적 콘텐츠 캐싱
  ↓
[Backend Service]
  ↓
[GKE Ingress / NEG (Network Endpoint Group)]
  ↓
[Pod]
```

**유형**
- Global HTTP(S) LB: L7, 글로벌 Anycast, SSL Termination
- Network LB (TCP/UDP): L4, 리전 단위
- Internal LB: VPC 내부 트래픽

---

## 8. IAM (Identity and Access Management)

**계층 구조**
```
Organization
  └── Folder
        └── Project
              └── Resource
```

**주요 개념**
- **Principal**: 누가 (Google Account, Service Account, Group)
- **Role**: 무엇을 할 수 있는가 (Basic / Predefined / Custom)
- **Binding**: Principal + Role을 Resource에 연결

```bash
# 서비스 계정 생성
gcloud iam service-accounts create game-server-sa

# 역할 부여
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:game-server-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# 키 생성 (Workload Identity 사용 시 불필요)
gcloud iam service-accounts keys create key.json \
  --iam-account=game-server-sa@PROJECT.iam.gserviceaccount.com
```

---

## 9. Cloud Monitoring & Logging

```bash
# 로그 조회 (gcloud)
gcloud logging read "resource.type=gke_container AND severity>=ERROR" \
  --limit=50 --format=json

# 로그 기반 메트릭 생성
gcloud logging metrics create error-rate \
  --description="Error log count" \
  --log-filter="severity=ERROR"
```

**모니터링 핵심 요소**
- **업타임 체크**: 엔드포인트 가용성 모니터링
- **알림 정책**: 메트릭 임계치 초과 시 PagerDuty, Slack 알림
- **대시보드**: CPU, 메모리, 연결 수, 레이턴시 시각화
- **SLO**: 서비스 수준 목표 설정 및 오류 예산 관리

---

## 10. VPC 네트워크

```bash
# VPC 생성
gcloud compute networks create game-vpc --subnet-mode=custom

# 서브넷 생성
gcloud compute networks subnets create game-subnet \
  --network=game-vpc \
  --region=asia-northeast3 \
  --range=10.0.0.0/24

# 방화벽 규칙
gcloud compute firewall-rules create allow-game-port \
  --network=game-vpc \
  --allow=tcp:8080 \
  --target-tags=game-server
```

**VPC Peering**: 서로 다른 VPC 간 내부 IP로 통신 (게임 서버 ↔ 공통 인프라)
**Cloud NAT**: Private VM이 인터넷 아웃바운드 가능 (인바운드 불가)

---

## 11. 자주 나오는 면접 질문

**Q. GKE Standard vs Autopilot?**
Standard: 노드 직접 관리, 더 많은 제어권. Autopilot: 노드 관리 불필요, Pod 단위 과금, 보안 강화 기본값.

**Q. Workload Identity란?**
K8s 서비스 계정을 GCP 서비스 계정에 매핑해 키 파일 없이 GCP API 접근. 키 노출 위험 제거.

**Q. 리전 vs 존(Zone)?**
리전: 지리적 위치 (asia-northeast3 = 서울). 존: 리전 내 독립 데이터센터 (asia-northeast3-a/b/c). 가용성을 위해 멀티존 배포.

**Q. Cloud SQL vs Cloud Spanner 선택 기준?**
Cloud SQL: 단일 리전, 기존 MySQL/PostgreSQL 마이그레이션. Spanner: 글로벌 분산, 강한 일관성, 수평 확장. 비용은 Spanner가 훨씬 비쌈.
