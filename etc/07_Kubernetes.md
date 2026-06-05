# Kubernetes 핵심 정리

---

## 1. 기본 개념

**클러스터 구성**
```
Control Plane (Master)          Worker Node
┌─────────────────────┐        ┌──────────────────┐
│ API Server          │        │ kubelet          │
│ etcd                │◄──────►│ kube-proxy       │
│ Scheduler           │        │ Container Runtime│
│ Controller Manager  │        │ (containerd)     │
└─────────────────────┘        └──────────────────┘
```

| 컴포넌트 | 역할 |
|---------|------|
| **API Server** | 모든 요청의 진입점. REST API |
| **etcd** | 클러스터 상태 저장 (Key-Value 스토어) |
| **Scheduler** | Pod를 어느 노드에 배치할지 결정 |
| **Controller Manager** | 원하는 상태 유지 (ReplicaSet, Deployment 등) |
| **kubelet** | 노드에서 Pod 실행/상태 보고 |
| **kube-proxy** | 노드 내 네트워크 규칙 관리 (Service → Pod 라우팅) |

---

## 2. 핵심 오브젝트

**Pod**: K8s 최소 배포 단위. 1개 이상의 컨테이너 포함. 같은 Pod 내 컨테이너는 localhost로 통신.

**Deployment**: Pod 복제본 관리, 롤링 업데이트, 롤백
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: game-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: game-server
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # 추가로 생성 가능한 Pod 수
      maxUnavailable: 0   # 업데이트 중 불가 Pod 수 (0 = 무중단)
  template:
    metadata:
      labels:
        app: game-server
    spec:
      containers:
      - name: game-server
        image: myrepo/game-server:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
```

**Service**: Pod에 안정적인 네트워크 엔드포인트 제공
```yaml
apiVersion: v1
kind: Service
metadata:
  name: game-server-svc
spec:
  selector:
    app: game-server
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP      # ClusterIP / NodePort / LoadBalancer
```

| Service 타입 | 설명 |
|-------------|------|
| ClusterIP | 클러스터 내부 접근만 가능 (기본값) |
| NodePort | 노드 IP:포트로 외부 접근 (30000~32767) |
| LoadBalancer | 클라우드 LB 프로비저닝. 외부 노출 |

**ConfigMap & Secret**
```yaml
# ConfigMap - 설정 데이터
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-config
data:
  DB_HOST: "mysql-svc"
  LOG_LEVEL: "info"

# Secret - 민감 데이터 (base64 인코딩)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQ=  # echo -n 'password' | base64
```

---

## 3. StatefulSet (게임 서버에서 중요)

게임 서버처럼 상태 유지가 필요하거나 고정 네트워크 ID가 필요한 경우 사용.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: battle-server
spec:
  serviceName: "battle-server"
  replicas: 3
  selector:
    matchLabels:
      app: battle-server
  template:
    ...
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

- Pod 이름 고정: `battle-server-0`, `battle-server-1`, `battle-server-2`
- DNS: `battle-server-0.battle-server.default.svc.cluster.local`
- 순서대로 생성/삭제

---

## 4. HPA (수평 자동 스케일링)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: game-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: game-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 5. Ingress

외부 트래픽을 내부 Service로 라우팅. L7 로드밸런서 역할.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: game-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: api.game.com
    http:
      paths:
      - path: /v1/auth
        pathType: Prefix
        backend:
          service:
            name: auth-svc
            port:
              number: 80
      - path: /v1/game
        pathType: Prefix
        backend:
          service:
            name: game-svc
            port:
              number: 80
```

---

## 6. kubectl 주요 명령어

```bash
# 조회
kubectl get pods -n game -o wide
kubectl get all -n game
kubectl describe pod game-server-xxx -n game
kubectl logs -f game-server-xxx -n game --previous  # 이전 컨테이너 로그
kubectl top pods -n game                             # 리소스 사용량

# 배포
kubectl apply -f deployment.yaml
kubectl rollout status deployment/game-server
kubectl rollout history deployment/game-server
kubectl rollout undo deployment/game-server --to-revision=2

# 디버깅
kubectl exec -it game-server-xxx -n game -- /bin/sh
kubectl port-forward pod/game-server-xxx 8080:8080  # 로컬에서 접근
kubectl cp game-server-xxx:/app/logs ./logs         # 파일 복사

# 스케일
kubectl scale deployment game-server --replicas=5
```

---

## 7. Pod 스케줄링 제어

```yaml
# nodeSelector: 특정 노드에만 배포
spec:
  nodeSelector:
    region: asia-northeast3

# affinity: 더 세밀한 스케줄링 규칙
spec:
  affinity:
    podAntiAffinity:  # 같은 앱 Pod는 다른 노드에 분산
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["game-server"]
        topologyKey: kubernetes.io/hostname

# taint/toleration: 특정 노드를 특정 Pod만 사용
# 노드에 taint 추가
kubectl taint nodes node1 dedicated=game:NoSchedule
# Pod에 toleration 추가
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "game"
    effect: "NoSchedule"
```

---

## 8. 자주 나오는 면접 질문

**Q. Deployment vs StatefulSet 언제 쓰나?**
Deployment: 상태 없는(Stateless) 앱 (API 서버, 웹서버). StatefulSet: 상태 있는 앱 (DB, 배틀 서버처럼 고정 ID 필요).

**Q. Readiness vs Liveness Probe 차이?**
- Liveness: 컨테이너가 살아있는지. 실패 시 Pod 재시작
- Readiness: 트래픽 받을 준비가 됐는지. 실패 시 Service에서 제외

**Q. Pod가 Pending 상태인 이유?**
리소스 부족(CPU/메모리), 노드 셀렉터 불일치, PVC 바인딩 실패 등. `kubectl describe pod`로 확인.

**Q. K8s에서 롤링 업데이트 무중단 배포 방법?**
`maxUnavailable: 0`, `maxSurge: 1` 설정 + readinessProbe 설정으로 새 Pod 준비 완료 후 구 Pod 제거.
