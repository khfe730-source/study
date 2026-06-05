# 게임 서버 프로그래머 필기시험 준비

**채용공고 기준**: 서버 프로그래머 (7년 이상) | PHP, Golang, MySQL, Linux, Docker | GCP, Kubernetes 우대

---

## 파일 목록

| 파일 | 분야 | 핵심 토픽 |
|------|------|----------|
| [01_OS_Linux.md](./01_OS_Linux.md) | OS / Linux | 프로세스/스레드, 메모리, 동기화, 스케줄링, epoll, 명령어 |
| [02_Database_MySQL.md](./02_Database_MySQL.md) | DB / MySQL | 인덱스, 트랜잭션, 격리수준, 락, InnoDB, 복제, 쿼리 최적화 |
| [03_Network.md](./03_Network.md) | Network | TCP/UDP, HTTP, TLS, 소켓, 웹소켓, 로드밸런싱 |
| [04_Golang.md](./04_Golang.md) | Golang | 고루틴, 채널, 동기화, GC, 인터페이스, context |
| [05_PHP.md](./05_PHP.md) | PHP | OOP, PDO, 세션, OPcache, Composer, 보안 |
| [06_Docker.md](./06_Docker.md) | Docker | 이미지/컨테이너, Dockerfile, Compose, 네트워크, 볼륨 |
| [07_Kubernetes.md](./07_Kubernetes.md) | Kubernetes | Pod/Deployment/Service, StatefulSet, HPA, kubectl |
| [08_GCP_Cloud.md](./08_GCP_Cloud.md) | GCP | GKE, Cloud SQL, Memorystore, Pub/Sub, IAM, VPC |
| [09_GameServer_Architecture.md](./09_GameServer_Architecture.md) | 게임 서버 | 동시성 제어, 세션, 실시간 동기화, 패킷 설계, 보안 |
| [10_DataStructure_Algorithm.md](./10_DataStructure_Algorithm.md) | 자료구조/알고리즘 | 배열, 해시맵, 트리, 힙, 그래프, 정렬, DP, 이진탐색 |
| [11_DesignPatterns.md](./11_DesignPatterns.md) | 디자인 패턴 | Singleton, Observer, Command, State, Strategy, Factory |
| [12_Redis.md](./12_Redis.md) | Redis 심화 | 자료구조, 분산락, Pub/Sub, Lua, Cluster, Sentinel, 캐시 전략 |

---

## 우선순위 공부 순서

1. **OS/Linux + Network** → 서버 개발 기본기. 거의 모든 필기에서 출제
2. **자료구조 & 알고리즘** → 코딩테스트와 겹침. 개념 문제로도 출제
3. **MySQL** → 트랜잭션, 인덱스, 락은 게임 서버 핵심
4. **Redis** → 게임 서버에서 필수. 분산 락, 랭킹, 캐시 전략
5. **Golang** → 주력 언어. 고루틴/채널 동작 원리 중요
6. **디자인 패턴** → 설계 문제, "이 상황에 어떤 패턴?" 유형
7. **Docker + Kubernetes** → 우대사항이지만 현업 필수
8. **GCP** → 우대사항. GKE, Memorystore 위주로
9. **게임 서버 아키텍처** → 실무 경험 기반 복습

---

## 자주 나오는 필기 유형

- **코드 출력 결과 예측** (Go/PHP 동시성, defer, 클로저)
- **SQL 쿼리 작성** (인덱스 활용, 트랜잭션)
- **시스템 설계** (CCU 10만 서버 설계, 재화 처리 동시성)
- **알고리즘** (정렬, 자료구조, 해시맵 활용)
- **네트워크 개념** (TCP 핸드쉐이크, HTTP 상태코드)
