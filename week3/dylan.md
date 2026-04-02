# 📘 Week 3 - Kubernetes 장애 대응을 위한 이론 정리

## 1. Kubernetes 전체 아키텍처

![Components of Kubernetes](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

### 🔵 Control Plane 구성요소

Control Plane은 클러스터의 두뇌 역할을 하며, 모든 결정을 내리고 상태를 관리한다.

| 컴포넌트 | 역할 |
|---|---|
| **kube-apiserver** | 모든 요청의 진입점. REST API를 통해 kubectl, 내부 컴포넌트와 통신 |
| **etcd** | 클러스터의 모든 상태(desired state)를 저장하는 분산 key-value 저장소 |
| **kube-scheduler** | 새로운 Pod를 어떤 Node에 배치할지 결정 |
| **kube-controller-manager** | Desired State와 Current State를 비교하고 일치시키는 루프 실행 |

### 🟢 Node 구성요소

Node는 실제 컨테이너가 실행되는 워커 머신이다.

| 컴포넌트 | 역할 |
|---|---|
| **kubelet** | Node에서 실행되는 에이전트. API Server로부터 Pod 명세를 받아 컨테이너를 실행/관리 |
| **container runtime** | 실제 컨테이너를 실행하는 엔진 (containerd, CRI-O 등) |
| **kube-proxy** | 각 Node에서 네트워크 규칙(iptables)을 관리하여 Service → Pod 트래픽을 라우팅 |

---

### ✅ 정상 흐름: Pod 생성 요청 시 내부 동작

```
① 사용자  →  kubectl apply -f pod.yaml
                    ↓
② kube-apiserver  →  요청 인증/인가 후 etcd에 Pod 명세 저장 (상태: Pending)
                    ↓
③ kube-scheduler  →  etcd의 변경을 감지 → Node 선택 → API Server를 통해 Node 할당 정보 업데이트
                    ↓
④ kubelet (해당 Node)  →  API Server로부터 자신에게 할당된 Pod 명세 수신
                    ↓
⑤ container runtime  →  kubelet 요청에 따라 이미지 pull & 컨테이너 실행
                    ↓
⑥ kubelet  →  Pod 상태를 API Server에 보고 (Running)
                    ↓
⑦ kube-proxy  →  Service 규칙에 따라 iptables 업데이트 → 트래픽 수신 가능
```

---

## 2. Pod Lifecycle & 상태

### 📊 Pod 상태 흐름

```
Pending  →  Running  →  Succeeded (정상 종료)
   ↓                 ↘
   ↓              Failed (비정상 종료)
   ↓
CrashLoopBackOff (반복 실패)
   
Running  →  Terminating  →  (삭제 완료)
```

### Pod 상태별 의미

| 상태 | 의미 | 주요 원인 |
|---|---|---|
| **Pending** | 스케줄링 대기 또는 이미지 Pull 중 | 리소스 부족, Node 없음, PVC 미연결 |
| **Running** | 컨테이너 실행 중 | - |
| **CrashLoopBackOff** | 컨테이너가 반복 실행 후 계속 실패 | 앱 에러, 설정 오류, OOM, Probe 실패 |
| **Terminating** | 삭제 요청 받아 종료 중 | Finalizer가 걸려 있으면 무한 대기 가능 |
| **OOMKilled** | 메모리 초과로 강제 종료 | Memory Limit 초과 |

---

### 🔍 Probe 개념

**livenessProbe**: 컨테이너가 살아있는지 확인. 실패 시 → 컨테이너 재시작

**readinessProbe**: 컨테이너가 트래픽을 받을 준비가 됐는지 확인. 실패 시 → Service의 Endpoints에서 제외 (트래픽 차단)

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10   # 컨테이너 시작 후 첫 체크까지 대기
  periodSeconds: 5           # 체크 주기
  failureThreshold: 3        # 몇 번 실패 시 재시작

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 3
```

---

### ✅ 정상 흐름: Pod 정상 기동

```
1. 컨테이너 시작
2. initialDelaySeconds 대기
3. readinessProbe 성공 → Endpoints에 Pod IP 등록 → 트래픽 수신 시작
4. livenessProbe 주기적 체크 → 성공하면 Running 유지
```

---

### 🚨 장애 상황 1: CrashLoopBackOff

**증상**: Pod가 계속 재시작되며 `CrashLoopBackOff` 상태

**장애 대응 순서**:
```bash
# 1. Pod 상태 및 재시작 횟수 확인
kubectl get pod <pod-name> -o wide

# 2. 현재 로그 확인
kubectl logs <pod-name>

# 3. 이전 컨테이너 로그 확인 (재시작 전 마지막 로그)
kubectl logs <pod-name> --previous

# 4. 상세 이벤트 확인
kubectl describe pod <pod-name>
# Events 섹션에서 OOMKilled, Error, Back-off 등 확인

# 5. 원인별 조치
# - OOMKilled → resources.limits.memory 조정
# - 설정 오류 → ConfigMap/Secret 확인
# - 이미지 에러 → 이미지 태그, 레지스트리 접근 확인
```

---

### 🚨 장애 상황 2: 서비스는 정상인데 트래픽이 안 붙는 경우

**원인**: readinessProbe 실패 → Endpoints에서 Pod IP 제거됨

```bash
# Endpoints 확인 (빈 경우 → readinessProbe 실패)
kubectl get endpoints <service-name>

# Pod의 readiness 상태 확인
kubectl describe pod <pod-name>
# Conditions 섹션의 Ready 항목 확인

# readinessProbe 실패 원인 확인
kubectl logs <pod-name>  # 앱 로그에서 /ready 엔드포인트 오류 확인
```

---

## 3. Kubernetes Networking

### 🌐 Service 종류

| 종류 | 접근 범위 | 사용 사례 |
|---|---|---|
| **ClusterIP** | 클러스터 내부에서만 접근 가능 | 서비스 간 내부 통신 (기본값) |
| **NodePort** | `<NodeIP>:<NodePort>`로 외부 접근 | 개발/테스트 환경 |
| **LoadBalancer** | 클라우드 LB를 통해 외부 접근 | 프로덕션 외부 노출 |
| **ExternalName** | 외부 DNS 이름으로 매핑 | 외부 서비스 연동 |

---

### 🔗 kube-proxy 역할 (iptables 기반)

kube-proxy는 각 Node에서 Service → Pod 트래픽을 라우팅하는 iptables 규칙을 관리한다.

```
Service(ClusterIP:Port) 요청
         ↓
iptables DNAT 규칙 (kube-proxy가 생성)
         ↓
실제 Pod IP:Port로 변환 & 전달
         ↓ (여러 Pod인 경우 라운드로빈)
Pod A or Pod B or Pod C
```

---

### ✅ 정상 흐름: Pod → Service → Pod 트래픽

```
Client Pod
   ↓  DNS 조회: my-service.default.svc.cluster.local → ClusterIP 획득
   ↓
kube-proxy iptables 규칙 적용 (DNAT)
   ↓
목적지 Pod로 패킷 전달
   ↓
응답 반환
```

---

### 🚨 장애 상황: Service 라우팅 실패

**증상**: `curl <ClusterIP>` 응답 없음 또는 Connection refused

**장애 대응 순서**:
```bash
# 1. Service 존재 및 ClusterIP 확인
kubectl get svc <service-name>

# 2. Endpoints 확인 (비어있으면 selector 문제)
kubectl get endpoints <service-name>
# 빈 경우 → Pod의 label이 Service의 selector와 일치하는지 확인

# 3. Pod label 확인
kubectl get pod --show-labels
kubectl describe svc <service-name>  # Selector 확인

# 4. Pod 직접 접근 테스트 (Service 우회)
kubectl exec -it <test-pod> -- curl <pod-ip>:<port>

# 5. kube-proxy 상태 확인
kubectl get pod -n kube-system | grep kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>

# 6. iptables 규칙 확인 (Node에서)
sudo iptables -t nat -L | grep <service-name>
```

**특정 노드에서만 통신이 안 되는 이유**: 해당 Node의 kube-proxy가 비정상 → iptables 규칙 미적용

---

## 4. Kubernetes Storage

### 💾 PV / PVC 개념

```
관리자         →  PersistentVolume(PV) 생성 (실제 스토리지 리소스)
                         ↕ 바인딩
개발자/앱      →  PersistentVolumeClaim(PVC) 생성 (스토리지 요청)
                         ↕
Pod            →  PVC를 Volume으로 마운트
```

| 리소스 | 설명 |
|---|---|
| **PV (PersistentVolume)** | 실제 스토리지를 추상화한 클러스터 리소스. NFS, EBS, GCE PD 등 |
| **PVC (PersistentVolumeClaim)** | 사용자가 스토리지를 요청하는 리소스. 용량, AccessMode 지정 |
| **StorageClass** | 동적 프로비저닝을 위한 스토리지 유형 정의. PVC 생성 시 자동으로 PV 생성 |

### AccessMode 종류

| 모드 | 의미 |
|---|---|
| `ReadWriteOnce (RWO)` | 단일 Node에서 읽기/쓰기 |
| `ReadOnlyMany (ROX)` | 여러 Node에서 읽기만 |
| `ReadWriteMany (RWX)` | 여러 Node에서 읽기/쓰기 (NFS 등) |

---

### ✅ 정상 흐름: Volume attach 흐름

```
1. StorageClass 정의 (프로비저너, 스토리지 유형)
2. PVC 생성 요청
3. StorageClass에 의해 자동으로 PV 생성 (동적 프로비저닝)
4. PVC ↔ PV 바인딩 (Bound 상태)
5. Pod에서 PVC를 Volume으로 마운트
6. kubelet이 Volume을 컨테이너에 attach & mount
```

---

### 🚨 장애 상황: PVC Pending 상태

**증상**: PVC가 `Pending` 상태에서 벗어나지 않음 → Pod도 `Pending`

**장애 대응 순서**:
```bash
# 1. PVC 상태 확인
kubectl get pvc
kubectl describe pvc <pvc-name>
# Events 섹션에서 오류 메시지 확인

# 2. 매칭되는 PV 확인 (정적 프로비저닝의 경우)
kubectl get pv
# PV의 StorageClass, AccessMode, 용량이 PVC와 일치하는지 확인

# 3. StorageClass 확인 (동적 프로비저닝의 경우)
kubectl get storageclass
kubectl describe storageclass <storageclass-name>
# 프로비저너가 제대로 설정되어 있는지 확인

# 4. 프로비저너 Pod 로그 확인
kubectl get pod -n kube-system | grep provisioner
kubectl logs -n kube-system <provisioner-pod>

# 5. Node의 스토리지 드라이버 상태 확인
kubectl get csidrivers
kubectl get csinode
```

**주요 원인**:
- 매칭되는 PV 없음 (용량, AccessMode, StorageClass 불일치)
- StorageClass가 존재하지 않음
- 클라우드 프로비저너 권한 부족
- 스토리지 용량 한도 초과

---

## 5. Workload 리소스

### 🔄 Deployment / ReplicaSet

```
Deployment
    └── ReplicaSet (현재 버전)
            ├── Pod 1
            ├── Pod 2
            └── Pod 3
    └── ReplicaSet (이전 버전, 롤백용 보존)
```

| 리소스 | 역할 |
|---|---|
| **Deployment** | Pod의 선언적 업데이트 관리. 롤링 업데이트, 롤백 담당 |
| **ReplicaSet** | 지정된 수의 Pod Replica가 항상 실행되도록 보장 |

---

### 💡 Desired State 개념

Kubernetes는 항상 **"선언된 상태(Desired State)"** 와 **"현재 상태(Current State)"** 를 비교하고 일치시키는 **Reconciliation Loop** 를 실행한다.

```
etcd에 저장된 Desired State
        ↕ 비교 (controller-manager)
현재 클러스터 Current State
        ↓ 차이 발생 시
자동 교정 동작 실행
```

---

### ✅ 정상 흐름: Rolling Update

```
1. kubectl apply -f deployment.yaml (새 이미지 버전 지정)
2. 새 ReplicaSet 생성
3. 새 ReplicaSet의 Pod를 1개씩 증가 (maxSurge)
4. readinessProbe 통과 확인
5. 기존 ReplicaSet의 Pod를 1개씩 감소 (maxUnavailable)
6. 모든 Pod 교체 완료 → 구 ReplicaSet은 replicas=0으로 보존 (롤백용)
```

---

### 🚨 장애 상황: Pod가 계속 재생성되는 이유

**이유**: ReplicaSet이 Desired 수를 유지하려 하기 때문

```bash
# 현재 ReplicaSet의 Desired/Current 상태 확인
kubectl get replicaset
kubectl describe replicaset <rs-name>

# Deployment 상태 확인
kubectl rollout status deployment/<deployment-name>

# 수동으로 삭제한 Pod가 다시 생기는 이유
# → ReplicaSet이 Desired 수 감지 → 새 Pod 생성 (정상 동작)
# Pod를 영구 삭제하려면 Deployment의 replicas를 0으로 조정

kubectl scale deployment <deployment-name> --replicas=0
```

**Rolling Update 중 Pod가 계속 Pending인 경우**:
```bash
# 업데이트 이력 확인
kubectl rollout history deployment/<deployment-name>

# 롤백
kubectl rollout undo deployment/<deployment-name>

# 특정 버전으로 롤백
kubectl rollout undo deployment/<deployment-name> --to-revision=2
```

---

## 6. 기본 디버깅 방법

### 🛠 핵심 kubectl 명령어

```bash
# ─────────────────────────────
# 상태 확인
# ─────────────────────────────
kubectl get pod -A                          # 모든 네임스페이스 Pod 목록
kubectl get pod <name> -o wide             # Node 정보 포함
kubectl get pod <name> -o yaml             # 전체 명세 확인

# ─────────────────────────────
# 로그 확인
# ─────────────────────────────
kubectl logs <pod-name>                    # 현재 컨테이너 로그
kubectl logs <pod-name> --previous         # 재시작 이전 로그
kubectl logs <pod-name> -c <container>     # 멀티 컨테이너 Pod
kubectl logs <pod-name> -f                 # 실시간 스트리밍

# ─────────────────────────────
# 상세 정보 & 이벤트
# ─────────────────────────────
kubectl describe pod <pod-name>            # 이벤트, 상태, 조건 확인
kubectl get events --sort-by='.lastTimestamp'  # 시간순 이벤트 확인
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>

# ─────────────────────────────
# 직접 접근 & 네트워크 테스트
# ─────────────────────────────
kubectl exec -it <pod-name> -- /bin/sh     # 컨테이너 내부 접속
kubectl exec -it <pod-name> -- curl <service>:<port>  # 내부 통신 테스트
kubectl port-forward pod/<pod-name> 8080:80  # 로컬에서 직접 접근
```

---

### 🔍 이벤트 기반 문제 추적

`kubectl describe`의 **Events 섹션**은 가장 중요한 진단 정보다.

```
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  5m     default-scheduler  0/3 nodes available: insufficient memory
  Warning  Failed            3m     kubelet            Failed to pull image: ErrImagePull
  Warning  BackOff           1m     kubelet            Back-off restarting failed container
```

| Event Type | 의미 |
|---|---|
| `FailedScheduling` | 스케줄링 실패. 리소스 부족, Taint, Affinity 문제 |
| `ErrImagePull` / `ImagePullBackOff` | 이미지 pull 실패. 이미지명, 레지스트리 인증 확인 |
| `OOMKilled` | 메모리 초과. Limit 조정 필요 |
| `FailedMount` | Volume 마운트 실패. PVC, StorageClass 확인 |
| `BackOff` | 컨테이너 재시작 반복. 앱 로그 확인 필요 |

---

### 🚨 kubectl이 정상 동작하지 않을 때

kubectl 자체가 안 될 때는 Node에 직접 접속하여 확인한다.

```bash
# ─────────────────────────────
# kubelet 상태 확인
# ─────────────────────────────
sudo systemctl status kubelet
sudo journalctl -u kubelet -f              # 실시간 kubelet 로그
sudo journalctl -u kubelet --since "10 minutes ago"

# ─────────────────────────────
# container runtime 상태 확인 (containerd)
# ─────────────────────────────
sudo systemctl status containerd
sudo crictl ps                             # 실행 중인 컨테이너 목록
sudo crictl logs <container-id>            # 컨테이너 로그

# ─────────────────────────────
# Control Plane Static Pod 확인
# ─────────────────────────────
ls /etc/kubernetes/manifests/              # Static Pod 파일 확인
sudo crictl ps | grep -E "apiserver|etcd|scheduler|controller"
```

---

## 7. k9s - 터미널 기반 Kubernetes UI

### 🖥 k9s란?

**k9s**는 터미널에서 실행되는 Kubernetes 클러스터 관리 TUI(Terminal User Interface) 도구다. `kubectl` 명령어를 일일이 타이핑하는 대신, 키보드만으로 클러스터 전체를 실시간으로 탐색하고 조작할 수 있다.

```
┌─────────────────────────────────────────────────────────────┐
│  Context: my-cluster          Namespace: default       <?>  │
├─────────────────────────────────────────────────────────────┤
│  NAMESPACE   NAME              READY  STATUS   RESTARTS AGE │
│  default     api-server-xxx    1/1    Running  0        2d  │
│  default     db-pod-yyy        0/1    Pending  0        5m  │  ← 선택
│  kube-system coredns-zzz       1/1    Running  0        2d  │
├─────────────────────────────────────────────────────────────┤
│  [l] Logs  [d] Describe  [e] Edit  [ctrl-d] Delete  [esc]  │
└─────────────────────────────────────────────────────────────┘
```

---

### ⚙️ 설치 방법

```bash
# macOS (Homebrew)
brew install derailed/k9s/k9s

# Linux (curl)
curl -sS https://webinstall.dev/k9s | bash

# Linux (패키지 직접 다운로드)
wget https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
tar -xzf k9s_Linux_amd64.tar.gz
sudo mv k9s /usr/local/bin/

# 버전 확인
k9s version
```

---

### 🚀 실행 방법

```bash
k9s                             # 기본 실행 (현재 kubeconfig context 사용)
k9s -n kube-system              # 특정 네임스페이스로 시작
k9s --context my-prod-cluster   # 특정 context로 시작
k9s --readonly                  # 읽기 전용 모드 (실수 방지)
```

---

### ⌨️ 핵심 단축키

#### 탐색

| 단축키 | 동작 |
|---|---|
| `:pod` | Pod 목록으로 이동 |
| `:svc` | Service 목록으로 이동 |
| `:deploy` | Deployment 목록으로 이동 |
| `:ns` | Namespace 목록으로 이동 |
| `:pvc` | PersistentVolumeClaim 목록 |
| `:node` | Node 목록으로 이동 |
| `:event` | Event 목록으로 이동 |
| `/` | 현재 목록에서 필터링 검색 |
| `esc` | 이전 화면으로 돌아가기 |

#### Pod 조작

| 단축키 | 동작 | kubectl 동등 명령 |
|---|---|---|
| `l` | 로그 보기 | `kubectl logs` |
| `p` | 이전 컨테이너 로그 | `kubectl logs --previous` |
| `s` | 컨테이너 쉘 접속 | `kubectl exec -it` |
| `d` | describe 보기 | `kubectl describe` |
| `e` | YAML 편집 | `kubectl edit` |
| `ctrl-d` | 리소스 삭제 | `kubectl delete` |
| `ctrl-k` | 강제 삭제 (force) | `kubectl delete --force` |

#### 화면 제어

| 단축키 | 동작 |
|---|---|
| `ctrl-a` | 모든 네임스페이스 토글 |
| `0`~`9` | 네임스페이스 즐겨찾기 전환 |
| `?` | 전체 단축키 도움말 |
| `q` | k9s 종료 |

---

### 🔍 장애 대응 시 k9s 활용 플로우

```
k9s 실행
   ↓
:pod → 비정상 Pod 확인 (색상으로 한눈에 식별)
   ↓
해당 Pod 선택 → d (describe) → Events 섹션 확인
   ↓
l (logs) → 앱 로그 확인
p (previous logs) → 재시작 이전 로그 확인
   ↓
:event → 클러스터 전체 이벤트 시간순 확인
   ↓
:node → Node 리소스 사용량 확인
```

**k9s의 색상 코드 (장애 식별에 유용)**:

| 색상 | 의미 |
|---|---|
| 🟢 초록 | 정상 (Running, Bound 등) |
| 🟡 노랑 | 주의 (Pending, Terminating 등) |
| 🔴 빨강 | 오류 (CrashLoopBackOff, Error, OOMKilled 등) |

---

### 💡 kubectl vs k9s 비교

| 상황 | kubectl | k9s |
|---|---|---|
| 특정 리소스 빠른 확인 | `kubectl get pod -A` | `:pod` + 즉시 확인 |
| 로그 확인 | `kubectl logs <pod> --previous` | Pod 선택 → `p` |
| 여러 네임스페이스 동시 확인 | 명령 반복 필요 | `ctrl-a`로 전체 토글 |
| 실시간 모니터링 | `--watch` 플래그 | 자동 실시간 갱신 |
| 스크립트/자동화 | ✅ 적합 | ❌ 부적합 |
| 장애 상황 빠른 탐색 | 명령 타이핑 필요 | ✅ 키보드 탐색으로 신속 대응 |

> **권장**: 평상시 모니터링과 장애 초기 탐색은 **k9s**, 스크립트 자동화와 정밀 조작은 **kubectl** 을 함께 사용하는 것이 효율적이다.

---

## 8. 장애 대응 체크리스트 요약

### 🗺 장애 발생 시 확인 순서 (Big Picture)

```
kubectl 동작 여부 확인
        ↓ 안 된다면
kubelet / API Server 확인 (journalctl, crictl)
        ↓ 된다면
kubectl get pod → 상태 확인
        ↓ 비정상이라면
kubectl describe pod → Events 섹션 분석
kubectl logs → 앱 로그 확인
kubectl get events → 클러스터 이벤트 확인
        ↓ 네트워크 문제라면
kubectl get endpoints → Service → Pod 연결 확인
kube-proxy 상태 확인
        ↓ 스토리지 문제라면
kubectl get pvc / pv → Binding 상태 확인
StorageClass / 프로비저너 확인
```

---

### 📋 주제별 장애 대응 한눈에 보기

| 영역 | 장애 증상 | 첫 번째 확인 명령 | 주요 의심 컴포넌트 |
|---|---|---|---|
| 아키텍처 | kubectl 응답 없음 | `systemctl status kubelet` | kubelet, API Server, etcd |
| Pod Lifecycle | CrashLoopBackOff | `kubectl logs --previous` | 앱 코드, 설정, 리소스 Limit |
| Pod Lifecycle | 트래픽 미연결 | `kubectl get endpoints` | readinessProbe, Service selector |
| Networking | Service 접근 불가 | `kubectl get endpoints` | kube-proxy, iptables, selector 불일치 |
| Storage | PVC Pending | `kubectl describe pvc` | StorageClass, PV 용량/AccessMode |
| Workload | Pod 계속 재생성 | `kubectl get replicaset` | ReplicaSet Desired, Deployment 설정 |

---

## 📎 참고: 자주 쓰는 디버깅 원라이너

```bash
# 모든 namespace에서 비정상 Pod 찾기
kubectl get pod -A --field-selector=status.phase!=Running

# 최근 이벤트 시간순 정렬
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# 특정 Pod의 Node 확인 후 해당 Node 상태 보기
NODE=$(kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}')
kubectl describe node $NODE

# 모든 Pod의 resource 사용량 확인
kubectl top pod -A

# Deployment 롤아웃 상태 모니터링
kubectl rollout status deployment/<name> --watch
```

---


---

## 9. 유형별 장애 대응 가이드

> 장애가 발생했을 때 **"어떤 증상인가"** 를 먼저 파악하고, 아래 유형 중 해당하는 항목을 찾아 의심 지점부터 순서대로 추적한다.

---

### 🔴 유형 1. Pod가 계속 재시작된다 (CrashLoopBackOff)

**증상**: `kubectl get pod` 에서 `CrashLoopBackOff` 또는 `RESTARTS` 수가 빠르게 증가

#### 의심 순서

```
1. 앱 자체 에러  →  kubectl logs --previous
2. 메모리 초과 (OOMKilled)  →  kubectl describe pod → Last State: OOMKilled
3. livenessProbe 설정 오류  →  kubectl describe pod → Events: Liveness probe failed
4. 설정 파일 누락  →  kubectl describe pod → Events: CreateContainerConfigError
5. 이미지 문제  →  kubectl describe pod → Events: ErrImagePull
```

```bash
# 재시작 이전 로그 확인 (가장 먼저)
kubectl logs <pod-name> --previous

# 상세 이벤트 확인
kubectl describe pod <pod-name>

# OOMKilled 여부 확인
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'

# ConfigMap/Secret 존재 여부 확인
kubectl get configmap
kubectl get secret
```

> 📌 **핵심**: 컨테이너가 시작에 실패하는 원인은 주로 앱 자체 에러, 잘못된 컨테이너 설정, 혹은 livenessProbe 반복 실패다. 재시작이 너무 빨라 로그를 못 보는 경우 `--previous` 플래그로 이전 컨테이너 로그를 확인한다.

> 📌 **OOMKilled**: Exit Code 137은 컨테이너가 메모리 초과로 종료됐음을 의미한다. Pod 이벤트를 통해 컨테이너 limit 초과인지, Node 전체의 메모리 과할당인지를 구분해야 한다.

---

### 🟡 유형 2. Pod가 Pending 상태에서 멈춰있다

**증상**: `kubectl get pod` 에서 `Pending` 상태가 계속 유지됨

#### 의심 순서

```
1. 리소스 부족 (CPU/Memory)  →  kubectl describe pod → FailedScheduling: insufficient
2. PVC 미연결  →  kubectl get pvc → Pending 또는 Unbound 상태
3. Node Selector / Affinity 불일치  →  kubectl describe pod → no nodes match node selector
4. Taint / Toleration 불일치  →  kubectl describe pod → had taints that pod didn't tolerate
5. 이미지 Pull 실패  →  kubectl describe pod → ErrImagePull / ImagePullBackOff
```

```bash
# 스케줄링 실패 원인 확인
kubectl describe pod <pod-name>
# → Events 섹션의 FailedScheduling 메시지 확인

# 노드 리소스 현황 확인
kubectl describe node <node-name>
kubectl top node

# PVC 상태 확인
kubectl get pvc
kubectl describe pvc <pvc-name>

# 이미지 수동 pull 테스트 (노드에서)
sudo crictl pull <image-name>
```

> 📌 **핵심**: Pending 상태의 가장 흔한 원인은 리소스 부족(CPU/메모리 고갈), 노드에 맞지 않는 hostPort 설정, 그리고 이미지 pull 실패다. `kubectl describe`의 Events 섹션에서 스케줄러 메시지를 확인하는 것이 첫 번째 단계다.

---

### 🟠 유형 3. Pod는 Running인데 트래픽이 안 된다

**증상**: Pod는 `Running` 상태지만 Service로 요청 시 응답 없음

#### 의심 순서

```
1. readinessProbe 실패  →  kubectl get endpoints → IP 목록 비어있음
2. Service selector와 Pod label 불일치  →  kubectl describe svc → Selector 확인
3. 포트 설정 오류  →  Service의 targetPort vs 컨테이너 port 불일치
4. NetworkPolicy에 의한 차단  →  kubectl get networkpolicy
5. kube-proxy 비정상  →  kubectl get pod -n kube-system | grep kube-proxy
```

```bash
# Endpoints 확인 (비어있으면 readinessProbe 또는 selector 문제)
kubectl get endpoints <service-name>

# Service selector와 Pod label 비교
kubectl describe svc <service-name>      # Selector 확인
kubectl get pod --show-labels            # Pod label 확인

# Pod 직접 접근 테스트 (Service 우회)
kubectl exec -it <debug-pod> -- curl <pod-ip>:<container-port>

# NetworkPolicy 확인
kubectl get networkpolicy -n <namespace>

# kube-proxy 로그 확인
kubectl logs -n kube-system <kube-proxy-pod>
```

> 📌 **핵심**: Endpoints가 비어있다면 Service의 selector와 Pod label이 일치하지 않는 것이 가장 흔한 원인이다. Service 설정의 selector label과 Pod의 실제 label을 비교해서 불일치를 찾아야 한다.

---

### 🔵 유형 4. Node가 NotReady 상태다

**증상**: `kubectl get nodes` 에서 `NotReady` 표시

#### 의심 순서

```
1. kubelet 비정상  →  systemctl status kubelet / journalctl -u kubelet
2. 디스크 압박 (DiskPressure)  →  kubectl describe node → Conditions
3. 메모리 압박 (MemoryPressure)  →  kubectl describe node → Conditions
4. 네트워크 단절  →  SSH 접근 가능 여부 확인
5. container runtime 장애  →  systemctl status containerd / crictl ps
```

```bash
# Node 상태 확인
kubectl get nodes
kubectl describe node <node-name>
# → Conditions 섹션: MemoryPressure, DiskPressure, PIDPressure 확인

# Node에 SSH 접속 후
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# 디스크 사용량 확인
df -h
du -sh /var/log/containers/*  # 컨테이너 로그 크기 확인

# container runtime 확인
sudo systemctl status containerd
sudo crictl ps
```

> 📌 **핵심**: NotReady의 주요 원인은 리소스 고갈, kubelet 장애, kube-proxy 장애다. `kubectl describe node`의 Conditions 섹션에서 `MemoryPressure`, `DiskPressure`, `PIDPressure` 중 어떤 조건이 `True`인지 확인한다.

> 📌 DiskPressure 상태인 경우, kubelet이 3일 이상 된 컨테이너 로그를 정리하면 디스크 공간을 확보하고 노드를 Ready 상태로 복구할 수 있다.

---

### 🟣 유형 5. 이미지를 못 가져온다 (ImagePullBackOff)

**증상**: `kubectl get pod` 에서 `ImagePullBackOff` 또는 `ErrImagePull`

#### 의심 순서

```
1. 이미지 이름/태그 오타  →  kubectl describe pod → Events에서 이미지명 확인
2. 프라이빗 레지스트리 인증 누락  →  imagePullSecrets 설정 확인
3. 레지스트리 접근 불가 (네트워크)  →  Node에서 직접 pull 시도
4. 이미지 태그 없음 (latest 등 부재)  →  레지스트리에서 태그 존재 확인
```

```bash
# 오류 메시지 확인
kubectl describe pod <pod-name>
# Events: Failed to pull image "xxx": rpc error...

# Node에서 직접 이미지 pull 테스트
sudo crictl pull <image:tag>

# imagePullSecret 존재 확인
kubectl get secret | grep regcred
kubectl get pod <pod-name> -o jsonpath='{.spec.imagePullSecrets}'
```

> 📌 **핵심**: ImagePullBackOff는 이미지 태그가 잘못됐거나, 레지스트리에 이미지가 없거나, 레지스트리 인증 실패, 또는 imagePullPolicy 설정 문제로 발생한다. 이 상태는 Pod가 무한정 재시도하는 것을 방지하기 위해 Kubernetes가 점진적으로 재시도 간격을 늘리는 back-off 메커니즘이다.

---

### ⚪ 유형 6. DNS 조회가 실패한다 (서비스 이름으로 접근 불가)

**증상**: Pod 내부에서 `nslookup <service-name>` 실패 또는 서비스 이름으로 연결 안 됨

#### 의심 순서

```
1. CoreDNS Pod 비정상  →  kubectl get pod -n kube-system | grep coredns
2. CoreDNS Endpoints 비어있음  →  kubectl get endpoints kube-dns -n kube-system
3. NetworkPolicy가 DNS 트래픽(UDP/53) 차단  →  kubectl get networkpolicy
4. Pod의 /etc/resolv.conf 잘못됨  →  kubectl exec → cat /etc/resolv.conf
5. 외부 DNS만 실패  →  CoreDNS forward 설정 확인
```

```bash
# CoreDNS Pod 상태 확인
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS 로그 확인
kubectl logs -n kube-system -l k8s-app=kube-dns

# Endpoints 확인 (비어있으면 CoreDNS Pod 장애)
kubectl get endpoints kube-dns -n kube-system

# 디버그 Pod로 DNS 직접 테스트
kubectl run dns-test --rm -it --restart=Never \
  --image=busybox:1.36 -- nslookup kubernetes.default

# Pod 내부 resolv.conf 확인
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
```

> 📌 **핵심**: CoreDNS Pod가 실행 중인지 먼저 확인하고, Endpoints가 비어있으면 CoreDNS Pod가 실패했거나 기본 환경에 배포되지 않은 것이다. 클러스터 내부 DNS 이름과 외부 DNS 중 어느 쪽이 실패하는지를 구분하는 것이 범위를 좁히는 핵심이다.

> 📌 특정 Pod만 DNS 실패가 발생한다면 해당 Pod의 DNS 설정을 확인해야 한다. 모든 Pod가 영향을 받는다면 CoreDNS 자체나 핵심 컴포넌트 장애를 먼저 의심한다.

---

### 🟤 유형 7. 볼륨 마운트 실패 / PVC Pending

**증상**: Pod가 `Pending` 상태이고, Events에 `FailedMount` 또는 PVC가 `Pending` 상태

#### 의심 순서

```
1. PVC와 매칭되는 PV 없음  →  kubectl get pv → AccessMode/용량/StorageClass 불일치
2. StorageClass 존재하지 않음  →  kubectl get storageclass
3. 동적 프로비저너 장애  →  kubectl get pod -n kube-system | grep provisioner
4. Node에서 볼륨 attach 실패  →  kubectl describe pod → FailedAttachVolume
5. 권한 문제 (클라우드 IAM)  →  프로비저너 Pod 로그 확인
```

```bash
# PVC 상태 확인
kubectl get pvc
kubectl describe pvc <pvc-name>   # Events: ProvisioningFailed 등

# PV 상태 확인
kubectl get pv

# StorageClass 확인
kubectl get storageclass
kubectl describe storageclass <sc-name>

# 프로비저너 Pod 로그 확인
kubectl get pod -n kube-system
kubectl logs -n kube-system <provisioner-pod>

# Volume mount 실패 상세 확인
kubectl describe pod <pod-name>
# → Events: FailedAttachVolume, FailedMount
```

---

### 🔶 유형 8. OOMKilled — 메모리 초과 종료

**증상**: Pod가 재시작되며 `kubectl describe pod`에서 `OOMKilled`, Exit Code `137`

#### 의심 순서

```
1. Memory Limit 너무 낮게 설정  →  resources.limits.memory 확인
2. 앱 메모리 누수  →  kubectl top pod로 메모리 증가 추이 확인
3. Node 전체 메모리 과할당  →  kubectl describe node → Allocated resources 확인
4. Java/JVM 힙 설정 문제  →  JVM 옵션에서 Xmx 확인
```

```bash
# OOMKilled 확인
kubectl describe pod <pod-name>
# Last State: Terminated Reason: OOMKilled Exit Code: 137

# Pod 메모리 사용량 확인
kubectl top pod <pod-name>

# Node 리소스 할당 현황 확인
kubectl describe node <node-name>
# → Allocated resources 섹션: 실제 사용 vs Limit 비교

# 메모리 limit 임시 조정
kubectl set resources deployment <deploy-name> \
  --limits=memory=512Mi
```

> 📌 **핵심**: OOMKilled는 Kubernetes가 아닌 Linux 커널의 OOM Killer 메커니즘이 컨테이너를 종료한 것이다. Exit Code 137로 기록된다. 컨테이너가 설정된 memory limit을 초과하거나, Node 전체가 메모리 압박을 받을 때 발생한다.

> 📌 가장 흔한 원인은 memory limit 잘못 설정과 memory request 과소 설정이다. 특히 Java 앱은 JVM 특성상 메모리 관리 요구사항이 달라 이 문제가 더 자주 발생한다.

---

### 🗺 유형별 빠른 참조표

| 증상 | 첫 번째 확인 | 주요 의심 원인 |
|---|---|---|
| CrashLoopBackOff | `kubectl logs --previous` | 앱 에러, OOMKilled, Probe 실패, 설정 누락 |
| Pod Pending | `kubectl describe pod` → Events | 리소스 부족, PVC 미연결, Taint, 이미지 오류 |
| 트래픽 미연결 | `kubectl get endpoints` | readinessProbe 실패, selector 불일치 |
| Node NotReady | `kubectl describe node` → Conditions | kubelet 장애, DiskPressure, MemoryPressure |
| ImagePullBackOff | `kubectl describe pod` → Events | 이미지명 오타, 인증 누락, 레지스트리 접근 불가 |
| DNS 실패 | `kubectl get pod -n kube-system` | CoreDNS 장애, NetworkPolicy 차단 |
| PVC Pending | `kubectl describe pvc` | PV 매칭 실패, StorageClass 없음, 프로비저너 장애 |
| OOMKilled | `kubectl describe pod` → Last State | Memory Limit 부족, 메모리 누수 |

---

### 📚 출처

| 출처 | URL | 설명 |
|---|---|---|
| Kubernetes 공식 문서 — Pod 디버깅 | https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/ | Pod 상태별 디버깅 공식 가이드 |
| Kubernetes 공식 문서 — 클러스터 디버깅 | https://kubernetes.io/docs/tasks/debug/debug-cluster/ | Node/클러스터 수준 디버깅 |
| Kubernetes 공식 문서 — DNS 디버깅 | https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/ | CoreDNS 디버깅 공식 가이드 |
| Kubernetes 공식 문서 — Pod 종료 원인 | https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/ | Pod 종료 메시지 확인 방법 |
| Komodor — Kubernetes Troubleshooting Guide | https://komodor.com/learn/kubernetes-troubleshooting-the-complete-guide/ | CrashLoopBackOff, NotReady 등 종합 가이드 |
| Komodor — OOMKilled (Exit Code 137) | https://komodor.com/learn/how-to-fix-oomkilled-exit-code-137/ | OOMKilled 원인 및 해결 |
| Komodor — Node NotReady | https://komodor.com/learn/how-to-fix-kubernetes-node-not-ready-error/ | Node NotReady 원인 분석 |
| groundcover — Node Not Ready | https://www.groundcover.com/kubernetes-troubleshooting/kubernetes-node-not-ready | NotReady 상태 상세 트러블슈팅 |
| PerfectScale — OOMKilled | https://www.perfectscale.io/blog/oomkilled | OOM 원인과 메모리 관리 가이드 |
| learnkube — Troubleshooting Deployments | https://learnkube.com/troubleshooting-deployments | Pod/Service/Ingress 시각적 가이드 |
| Spacelift — Kubernetes Troubleshooting | https://spacelift.io/blog/kubernetes-troubleshooting | 전체 트러블슈팅 전략 (2026 업데이트) |
| SUSE Blog — Total Troubleshooting Guide | https://www.suse.com/c/observability-the-total-kubernetes-troubleshooting-guide/ | ImagePullBackOff, CrashLoopBackOff 가이드 |
| Middleware.io — Top 10 Techniques | https://middleware.io/blog/kubernetes-troubleshooting-techniques/ | Node/Pod/Service 디버깅 10가지 기법 |
| Container Solutions — DNS Runbook | https://containersolutions.github.io/runbooks/posts/kubernetes/dns-failures/ | DNS 장애 단계별 runbook |
| OneUptime — CoreDNS Debug Guide | https://oneuptime.com/blog/post/2026-01-19-kubernetes-coredns-troubleshooting-guide/view | CoreDNS 디버깅 종합 가이드 |
| Google Cloud — Node NotReady (GKE) | https://docs.cloud.google.com/kubernetes-engine/docs/troubleshooting/node-notready | GKE 기반 NotReady 심화 가이드 |
| AWS re:Post — EKS DNS 트러블슈팅 | https://repost.aws/knowledge-center/eks-dns-failure | EKS 환경 DNS 장애 대응 |



> 🔥 **핵심 마인드셋**: Kubernetes의 모든 장애는 결국 "어느 컴포넌트가 어느 상태 전환에 실패했는가"의 문제다. 항상 **이벤트 → 로그 → 컴포넌트 상태** 순서로 추적하자.