# Docker 핵심 정리

---

## 1. Docker 기본 개념

| 개념 | 설명 |
|------|------|
| **이미지 (Image)** | 컨테이너 실행에 필요한 파일 시스템 스냅샷. 불변(Immutable) |
| **컨테이너 (Container)** | 이미지의 실행 인스턴스. 격리된 프로세스 |
| **레이어 (Layer)** | 이미지는 읽기 전용 레이어들의 스택. 변경 사항은 새 레이어 추가 |
| **레지스트리 (Registry)** | 이미지 저장소. Docker Hub, GCR, ECR 등 |
| **볼륨 (Volume)** | 컨테이너 외부의 데이터 영속성 저장소 |

**VM vs 컨테이너**
- VM: 하이퍼바이저 위에 전체 OS. 무겁고 느린 부팅
- 컨테이너: 호스트 OS 커널 공유, 프로세스 격리. 가볍고 빠름 (초 단위 부팅)

---

## 2. Docker 핵심 명령어

```bash
# 이미지
docker build -t myapp:1.0 .          # 빌드
docker pull nginx:alpine              # 가져오기
docker push myrepo/myapp:1.0          # 푸시
docker images                         # 이미지 목록
docker rmi nginx:alpine               # 이미지 삭제
docker image prune                    # 미사용 이미지 정리

# 컨테이너 실행
docker run -d --name myapp \
  -p 8080:80 \                        # 호스트:컨테이너 포트 매핑
  -e ENV=production \                 # 환경변수
  -v /host/data:/app/data \           # 볼륨 마운트
  --memory="512m" \                   # 메모리 제한
  --cpus="1.5" \                      # CPU 제한
  myapp:1.0

# 컨테이너 관리
docker ps                             # 실행 중 컨테이너
docker ps -a                          # 전체 컨테이너
docker stop/start/restart myapp
docker rm myapp                       # 컨테이너 삭제
docker logs -f myapp                  # 로그 실시간 확인
docker exec -it myapp /bin/sh         # 컨테이너 내부 접속
docker inspect myapp                  # 상세 정보
docker stats                          # 리소스 사용량 실시간
```

---

## 3. Dockerfile 작성

```dockerfile
# 멀티 스테이지 빌드 (이미지 크기 최소화)
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# 최종 실행 이미지
FROM alpine:3.19
RUN apk --no-cache add ca-certificates tzdata
WORKDIR /app
COPY --from=builder /app/server .
COPY --from=builder /app/config ./config

# 보안: root 대신 일반 사용자 실행
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:8080/health || exit 1
ENTRYPOINT ["/app/server"]
```

**레이어 캐싱 최적화 원칙**
1. 변경 빈도가 낮은 명령을 먼저 (의존성 설치 → 소스 복사 순서)
2. `COPY go.mod go.sum ./` + `RUN go mod download` 별도 레이어
3. `.dockerignore`로 불필요한 파일 제외

```
# .dockerignore
.git
*.md
*.test
vendor/
```

---

## 4. Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  game-server:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=mysql
      - REDIS_HOST=redis
    depends_on:
      mysql:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - game-net

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: gamedb
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - game-net

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - game-net

volumes:
  mysql_data:
  redis_data:

networks:
  game-net:
    driver: bridge
```

```bash
# 주요 명령
docker compose up -d        # 백그라운드 시작
docker compose down -v      # 중지 및 볼륨 삭제
docker compose logs -f      # 전체 로그
docker compose ps           # 상태 확인
docker compose exec game-server /bin/sh  # 컨테이너 접속
```

---

## 5. 네트워크

| 네트워크 타입 | 설명 |
|-------------|------|
| bridge | 기본. 컨테이너 간 통신. 호스트와 격리 |
| host | 호스트 네트워크 직접 사용. 성능 좋지만 격리 없음 |
| none | 네트워크 없음 |
| overlay | Swarm/K8s에서 다중 호스트 간 통신 |

- 같은 네트워크의 컨테이너는 **컨테이너 이름**으로 통신
- 외부 노출은 `-p` 또는 `ports` 설정

---

## 6. 볼륨 (Volume) vs 바인드 마운트

| 항목 | Volume | Bind Mount |
|------|--------|------------|
| 관리 | Docker가 관리 | 호스트 경로 직접 지정 |
| 위치 | `/var/lib/docker/volumes/` | 호스트 임의 경로 |
| 이식성 | 높음 | 낮음 |
| 성능 | 더 나음 | 호스트 의존 |
| 사용 | 데이터 영속성 | 개발 시 코드 마운트 |

---

## 7. 보안 베스트 프랙티스

```dockerfile
# 1. non-root 사용자
USER appuser

# 2. 최소 권한 베이스 이미지
FROM alpine:3.19  # distroless도 고려

# 3. 멀티 스테이지로 빌드 도구 제거

# 4. 읽기 전용 파일 시스템
docker run --read-only myapp

# 5. 리소스 제한
docker run --memory="512m" --cpus="1"

# 6. 시크릿은 환경변수보다 Docker Secret 사용
docker secret create db_password secret.txt
```

---

## 8. 이미지 크기 최적화

```bash
# 이미지 레이어 분석
docker history myapp:1.0
dive myapp:1.0  # dive 도구

# 최소 베이스 이미지 선택 (크기 비교)
ubuntu:22.04    ~77MB
debian:slim     ~75MB
alpine:3.19     ~7MB
scratch         0MB (정적 바이너리용)
distroless      ~2MB
```

---

## 9. 자주 나오는 면접 질문

**Q. ENTRYPOINT vs CMD 차이?**
- `CMD`: 기본 명령. `docker run` 시 인자로 override 가능
- `ENTRYPOINT`: 고정 실행파일. CMD는 인자로 전달됨
- 보통 `ENTRYPOINT ["./server"]` + `CMD ["--config=prod.yaml"]` 조합

**Q. 컨테이너가 삭제되면 데이터가 사라지는 이유?**
컨테이너 레이어는 임시(ephemeral). 영속성 필요 시 볼륨이나 바인드 마운트 사용.

**Q. Docker 이미지 빌드 캐시가 무효화되는 시점?**
해당 레이어 또는 그 이전 레이어의 명령이나 컨텍스트 파일이 변경될 때.

**Q. 컨테이너의 PID 1 문제?**
PID 1 프로세스는 시그널 핸들러를 명시적으로 구현해야 함. Shell form(`CMD server`)은 sh가 PID 1이 되어 SIGTERM이 자식 프로세스에 전달 안 됨. Exec form(`CMD ["server"]`) 사용 권장.
