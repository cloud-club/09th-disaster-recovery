# Kubernetes 장애 대응 이론 정리

## 1️⃣ Kubernetes 전체 아키텍처

### Control Plane
Control Plane은 클러스터 전체를 관리하고 조율하는 역할.

* **kube-apiserver**: 클러스터에서 모든 요청의 단일 진입점. (클러스터 상태를 읽고 쓰는 유일한 창구)
    * 모든 컴포넌트는 서로 직접 통신하지 않고 반드시 kube-apiserver를 통해 상태를 주고받음. kubectl / 내부 컴포넌트 모두 여기로 통신.
    * 인증/인가, 요청 검증 후 etcd에 상태 저장 

* **etcd**: 클러스터의 모든 상태를 저장하는 분산 Key-Value 저장소 (단일 source of truth)
    * Pod, Service, Node 등 모든 상태가 저장됨

* **scheduler** : Pending 상태 Pod을 실행할 Node 선택
    * Pod → Node 매핑 결정 (Pending 상태 Pod을 어떤 Node에 배치할지 결정)
    * 노드 평가 기준: 가용 리소스, Node Affinity, Taint/Toleration 

* **controller-manager** : 다양한 컨트롤러들의 집합 (Deployment, ReplicaSet 등)
    * 현재 상태와 원하는 상태(desired state)를 계속 비교
    * 상태를 원하는 방향으로 계속 맞춤 (예: Pod 부족하면 새로 생성)

---

### Node
Node는 실제로 컨테이너가 실행되는 워커 머신.

* **kubelet**: 각 노드에서 동작하는 에이전트로, Pod 실행 및 상태 관리 담당
    - apiserver로부터 할당된 Pod 명세 감지 → container runtime에 컨테이너 실행 지시
    - 컨테이너 상태를 주기적으로 apiserver에 보고
    - liveness/readiness probe 실행 및 결과 반영

* **container runtime**: 실제 컨테이너를 실행하는 소프트웨어
    - 컨테이너 이미지 pull, 컨테이너 생성/시작/중지 담당
    - 종류: containerd, CRI-O 등

* **kube-proxy**: 각 노드에서 네트워크(iptables) 규칙을 관리하는 컴포넌트
    - Service 및 Endpoints 변경 감지 → iptables 규칙 업데이트
    - Service IP로 들어오는 트래픽을 실제 Pod IP로 전달
    - 노드 단위로 동작하므로 특정 노드의 kube-proxy 장애 시 해당 노드에서만 통신 불가

---

### Pod 생성 요청 시 내부 동작 흐름

1. **[요청]** kubectl apply → kube-apiserver로 요청 전달
2. **[검증/저장]** kube-apiserver → 인증/인가/유효성 검사 후 etcd에 Pod 객체 저장 (상태: Pending)
3. **[스케줄링]** scheduler → Pending Pod 감지 → 노드 평가 → 배치 노드 결정 → apiserver를 통해 etcd에 기록
4. **[실행 지시]** kubelet → 자신에게 할당된 Pod 감지 → container runtime에 컨테이너 실행 지시
5. **[컨테이너 실행]** container runtime → 이미지 pull → 컨테이너 실행
6. **[상태 보고]** kubelet → 컨테이너 상태 확인 후 apiserver에 Running 상태 보고
7. **[네트워크 연결]** kube-proxy → Service 객체 감지 → iptables 규칙 업데이트 → 트래픽 전달 가능 상태

→ 핵심: **apiserver 중심으로 상태 저장 → scheduler가 배치 → kubelet이 실행**


---

## 2️⃣ Pod Lifecycle & 상태

### Pod의 주요 상태
- **Pending**: 스케줄러가 아직 노드에 Pod를 배치하지 못한 상태
    - 원인: 리소스 부족, Node Selector/Affinity 불일치, PVC 미바인딩

- **Running**: 컨테이너가 하나 이상 실행 중인 상태
    - Running이어도 readinessProbe 실패 시 트래픽 미수신 가능

- **CrashLoopBackOff**: 컨테이너가 반복 실행 → 반복 크래시 → 재시작 대기 간격 증가
    - 재시작 간격: 10s → 20s → 40s → 최대 5분으로 지수 증가
    - 원인: 
        - 애플리케이션 예외/패닉으로 프로세스 종료
        - 필수 환경변수·ConfigMap·Secret 누락
        - livenessProbe 실패로 kubelet이 강제 재시작
        - OOMKilled (메모리 한도 초과)
    - 진단: kubectl logs --previous, kubectl describe pod

- **Terminating**: 삭제 요청 후 graceful shutdown 진행 중인 상태
    - terminationGracePeriodSeconds (기본 30s) 내 종료 안 되면 SIGKILL
    - finalizer가 남아있으면 무한 Terminating 가능 → kubectl patch로 제거


### Probe의 역할

Probe는 Kubernetes가 컨테이너의 상태를 능동적으로 확인하는 메커니즘.

- **livenessProbe**: 컨테이너가 살아있는지 확인
    - 실패 시 kubelet이 컨테이너 재시작
    - 용도: 데드락·무한루프 등 응답 불가 상태 감지
    - 주의: 너무 민감하게 설정 시 불필요한 재시작 유발

- **readinessProbe**: 컨테이너가 트래픽을 받을 준비가 됐는지 확인
    - 실패 시 Endpoints에서 Pod IP 제거 → 서비스 트래픽 차단
    - 성공 시 Endpoints에 복귀
    - 용도: 앱 초기화 완료 전 트래픽 차단, 일시적 과부하 시 트래픽 격리


📌 학습 포인트

- CrashLoopBackOff 발생 이유
    - 컨테이너 entrypoint 프로세스가 exit code 0이 아닌 값으로 종료
    - 환경변수, Secret, ConfigMap 마운트 실패
    - livenessProbe failureThreshold 초과
    - 메모리 초과로 OOMKilled → 재시작 반복

- 서비스는 정상인데 트래픽이 안 붙는 경우
    - readinessProbe 실패 → Pod가 Endpoints에서 제외됨
    - Pod의 label이 Service의 selector와 불일치
    - targetPort와 컨테이너 실제 포트 불일치
    - NetworkPolicy가 해당 Pod로의 ingress 차단
    - Pod가 Running이지만 앱이 아직 초기화 중 (startup 시간 부족)

---

## 3️⃣ Kubernetes Networking

### Service 종류

- **ClusterIP**: 클러스터 내부에서만 접근 가능한 가상 IP 할당
    - 기본 Service 타입
    - Pod끼리 통신 시 사용 (외부 노출 없음)
    - DNS: `<service>.<namespace>.svc.cluster.local`

- **NodePort**: ClusterIP 기능 포함 + 모든 노드의 특정 포트(30000~32767)로 외부 노출
    - 요청 흐름: `외부 → NodeIP:NodePort → ClusterIP → Pod`
    - 노드 IP가 바뀌면 접근 경로도 바뀌는 단점

- **LoadBalancer**: NodePort 기능 포함 + 클라우드 LB 자동 프로비저닝
    - 클라우드 환경(AWS ELB, GCP LB 등)에서 주로 사용
    - 온프레미스에서는 MetalLB 등 별도 구현 필요

---

### kube-proxy 역할 (iptables 기반)

- 각 노드에 DaemonSet으로 실행
- Service → Pod IP 매핑을 iptables 규칙으로 관리
- Service의 ClusterIP로 들어온 패킷을 실제 Pod IP로 DNAT
- Endpoints 변경(Pod 추가/제거) 시 iptables 규칙 자동 갱신
- 로드밸런싱: iptables의 statistic 모듈로 랜덤 분산 (라운드로빈 아님)
- 모드: iptables(기본) / ipvs(대규모 클러스터에서 성능 우위)

---

### Pod → Service → Pod 트래픽 흐름

- **1단계 - DNS 조회**: Pod A가 Service DNS 조회 → CoreDNS가 ClusterIP 반환
- **2단계 - 패킷 가로채기**: 패킷이 ClusterIP로 전송되면 iptables PREROUTING 체인에서 감지
- **3단계 - DNAT**: Endpoints 중 하나의 Pod IP로 목적지 변환
- **4단계 - 수신**: Pod B는 출발지가 Pod A IP인 패킷 수신
- **5단계 - 응답**: 응답 패킷은 conntrack으로 역방향 SNAT 처리

---

### 📌 학습 포인트

#### Service를 통해 트래픽이 전달되는 과정
- CoreDNS → ClusterIP 반환
- iptables PREROUTING/OUTPUT 체인에서 ClusterIP 패킷 감지
- DNAT 규칙으로 Endpoints 중 하나의 Pod IP로 변환
- 응답 패킷은 conntrack으로 역방향 SNAT 처리
- Pod 추가/삭제 시 kube-proxy가 Endpoints 감지 → iptables 규칙 갱신

#### 특정 노드에서만 통신이 안 되는 이유
- 해당 노드의 kube-proxy 다운 → iptables 규칙 미갱신
- 노드의 iptables 규칙 꼬임 또는 누락 → `iptables -L -n -t nat`으로 확인
- CNI 플러그인 오작동 → Pod 간 오버레이 네트워크 단절
- NetworkPolicy가 특정 노드의 Pod에만 적용된 경우
- 노드 간 라우팅 경로 문제 (MTU 불일치 포함)
- 진단 순서: `kubectl get endpoints` → `iptables -L` → CNI 로그 확인

---

## 4️⃣ Kubernetes Storage

### PV / PVC 개념

- **PV (PersistentVolume)**: 클러스터 관리자가 미리 프로비저닝한 실제 스토리지 리소스
    - 클러스터 레벨 리소스 (네임스페이스 미소속)
    - NFS, EBS, GCE PD 등 다양한 백엔드 지원
    - Reclaim Policy: Retain / Delete / Recycle

- **PVC (PersistentVolumeClaim)**: 사용자가 스토리지를 요청하는 오브젝트
    - 네임스페이스 레벨 리소스
    - 요청 조건(용량, AccessMode)에 맞는 PV와 바인딩
    - AccessMode 종류:
        - `ReadWriteOnce` (RWO): 단일 노드에서 읽기/쓰기
        - `ReadOnlyMany` (ROX): 다수 노드에서 읽기 전용
        - `ReadWriteMany` (RWX): 다수 노드에서 읽기/쓰기

---

### StorageClass

- **역할**: PVC 요청 시 PV를 동적으로 프로비저닝하는 템플릿
- **동작**: PVC 생성 → StorageClass의 provisioner가 PV 자동 생성 → 바인딩
- **주요 필드**:
    - `provisioner`: 스토리지 백엔드 드라이버 (예: `ebs.csi.aws.com`)
    - `reclaimPolicy`: PVC 삭제 시 PV 처리 방식
    - `volumeBindingMode`:
        - `Immediate`: PVC 생성 즉시 바인딩
        - `WaitForFirstConsumer`: Pod 스케줄링 후 바인딩 (노드 토폴로지 고려)

---

### Volume Attach 흐름

- **1단계 - PVC 생성**: 사용자가 PVC 생성 → StorageClass로 PV 동적 프로비저닝
- **2단계 - 바인딩**: PV ↔ PVC 바인딩 완료 → 상태: `Bound`
- **3단계 - Pod 스케줄링**: Pod가 노드에 배치됨
- **4단계 - Attach**: 해당 노드에 볼륨 연결 (클라우드의 경우 EBS attach 등)
- **5단계 - Mount**: kubelet이 노드 내 디렉토리에 볼륨 마운트
- **6단계 - 컨테이너 기동**: 컨테이너가 마운트된 경로로 볼륨 접근

---

### 📌 학습 포인트

#### PVC Pending 상태의 원인
- 조건에 맞는 PV가 없음 (용량, AccessMode 불일치)
- StorageClass가 존재하지 않거나 provisioner 미설치
- `volumeBindingMode: WaitForFirstConsumer`인데 Pod가 아직 미스케줄링
- 클라우드 프로바이더 권한 부족으로 동적 프로비저닝 실패
- 진단: `kubectl describe pvc` → Events 확인

#### 스토리지 문제와 Pod 생성 실패의 관계
- PVC가 `Pending`이면 Pod도 `Pending` 상태로 대기
- 볼륨 Attach 실패 시 Pod가 노드에 배치되어도 컨테이너 미기동
- 이전 Pod의 볼륨이 다른 노드에 Attach된 채로 남아있으면 신규 Pod가 다른 노드에 스케줄링 불가 (RWO 제약)
- 진단 순서: `kubectl describe pod` → `FailedAttachVolume` / `FailedMount` 이벤트 확인

---

## 5️⃣ Workload 리소스

---

### Deployment / ReplicaSet

- **ReplicaSet**: 지정한 수의 Pod 복제본을 항상 유지하는 컨트롤러
    - `spec.replicas`로 원하는 Pod 수 선언
    - Pod 삭제 감지 시 즉시 재생성

- **Deployment**: ReplicaSet을 관리하는 상위 컨트롤러
    - 롤링 업데이트 / 롤백 기능 제공
    - 이미지 변경 시 새 ReplicaSet 생성 → 점진적 교체
    - `kubectl rollout undo`로 이전 ReplicaSet으로 롤백

---

### Desired State 개념

- Kubernetes의 핵심 원칙: **선언형(Declarative) 관리**
- 사용자가 원하는 상태(Desired State)를 선언 → 컨트롤러가 현재 상태와 비교 → 차이를 지속적으로 조정
- 이 루프를 **Reconciliation Loop** (조정 루프)라고 함
- ReplicaSet이 Pod를 감시하다가 수가 줄면 → 즉시 재생성하는 것도 이 원칙의 결과

---

### Rolling Update 동작 방식

- **maxUnavailable**: 업데이트 중 동시에 내릴 수 있는 Pod 최대 수 (기본값: 25%)
- **maxSurge**: 원하는 수 초과로 동시에 띄울 수 있는 Pod 최대 수 (기본값: 25%)
- 동작 순서:
    - 새 ReplicaSet 생성 → 새 Pod 점진적 증가
    - 구 ReplicaSet Pod 점진적 감소
    - 구 ReplicaSet은 삭제되지 않고 replicas=0으로 보존 (롤백용)
- `kubectl rollout status deployment/<name>`으로 진행 상태 확인

---

### 📌 학습 포인트

#### Pod가 계속 재생성되는 이유
- ReplicaSet의 Reconciliation Loop가 항상 Desired 수를 유지하려 함
- Pod가 CrashLoopBackOff로 죽으면 → ReplicaSet이 새 Pod 재생성 → 반복
- livenessProbe 실패로 kubelet이 재시작 → 크래시 반복 시 위와 동일
- 해결 없이 Pod만 삭제해도 ReplicaSet이 즉시 재생성

#### 수동으로 삭제한 Pod가 다시 생성되는 이유
- Pod는 ReplicaSet의 관리 대상
- 삭제 시 ReplicaSet이 현재 수(Actual) < 원하는 수(Desired) 감지
- Reconciliation Loop가 즉시 새 Pod 생성
- Pod를 영구 삭제하려면 Deployment/ReplicaSet의 `replicas: 0` 또는 리소스 자체를 삭제해야 함

---

## 6️⃣ 기본 디버깅 방법

**`kubectl logs`** : Pod 내 컨테이너의 stdout/stderr 출력 확인
- 주요 옵션:
    - `-f`: 실시간 로그 스트리밍
    - `--previous` / `-p`: 이전 크래시 컨테이너 로그 확인 (CrashLoopBackOff 진단 시 필수)
    - `-c <container>`: 멀티 컨테이너 Pod에서 특정 컨테이너 지정
    - `--tail=N`: 최근 N줄만 출력
- 한계: Pod가 Pending 상태면 로그 없음 → `kubectl describe`로 전환

**`kubectl describe`** : 리소스의 상세 정보 + Events 섹션 확인
- 확인 가능한 정보:
    - Pod 스케줄링 실패 원인 (Insufficient CPU/Memory 등)
    - Volume mount 실패 (`FailedMount`, `FailedAttachVolume`)
    - Probe 실패 내역
    - 컨테이너 재시작 횟수 및 종료 코드
- Events는 기본 1시간 보존 → 오래된 문제는 누락 가능

**`kubectl get events`** : 클러스터 전체 또는 특정 네임스페이스의 이벤트 목록 확인
- 주요 옵션:
    - `--sort-by='.lastTimestamp'`: 시간순 정렬
    - `--field-selector involvedObject.name=<pod명>`: 특정 리소스 필터링
    - `-n <namespace>`: 네임스페이스 지정
- `kubectl describe`의 Events보다 넓은 범위 확인 가능
- Warning 타입 이벤트에 집중 → 문제 원인 빠르게 파악

---

### 📌 학습 포인트

#### 이벤트 기반 문제 추적 방법
- `kubectl get events --sort-by='.lastTimestamp'`로 최신 이벤트부터 확인
- Warning 이벤트 → Reason 필드로 원인 분류
    - `FailedScheduling`: 스케줄링 실패 (리소스/어피니티 문제)
    - `FailedMount`: 볼륨 마운트 실패
    - `BackOff`: 컨테이너 재시작 반복
    - `Unhealthy`: Probe 실패
- 이벤트 → describe → logs 순서로 범위를 좁혀가며 추적
- 이벤트 보존 시간(1시간) 초과 시 → kubelet 로그로 보완

#### kubectl이 정상 동작하지 않을 때 확인해야 할 위치

- **kubelet 로그**
    - kubelet이 죽으면 Pod 생성/삭제/Probe 모두 동작 안 함
    - 확인: `journalctl -u kubelet -f` 또는 `systemctl status kubelet`
    - 주요 원인: 인증서 만료, 디스크 풀, 설정 오류

- **container runtime 로그**
    - kubelet이 컨테이너 생성을 runtime에 위임 → runtime 장애 시 컨테이너 미기동
    - containerd 확인: `journalctl -u containerd -f`
    - 주요 원인: 이미지 pull 실패, 소켓 연결 오류, 디스크 공간 부족

- **journalctl**
    - systemd 기반 로그 통합 조회 도구
    - 주요 사용법:
        - `journalctl -u kubelet --since "10 minutes ago"`: 최근 10분 kubelet 로그
        - `journalctl -u containerd -n 100`: containerd 최근 100줄
        - `journalctl -p err -xe`: 에러 레벨 이상 로그만 필터링

---

## 🧠 과제 
### 시나리오: Pod가 뜨지 않는다

---

#### 1. 정상 흐름 설명
- `kubectl apply` → API Server가 Deployment 스펙을 etcd에 저장
- Deployment 컨트롤러 → ReplicaSet 생성
- ReplicaSet 컨트롤러 → Pod 생성 (상태: `Pending`)
- 스케줄러 → 조건에 맞는 노드 배정
- kubelet → PVC 마운트 → container runtime에 컨테이너 생성 요청
- 컨테이너 기동 → livenessProbe / readinessProbe 통과
- readinessProbe 성공 → Endpoints에 Pod IP 등록 → 트래픽 수신

---

#### 2. 장애 상황 정의

- `kubectl get pod` 결과가 `Pending` 또는 `CrashLoopBackOff` 또는 계속 `Terminating`
- 배포한 지 한참 지났는데 Pod가 Running이 되지 않는 상태

---

#### 3. 장애 대응 방법

**Step 1 - 상태 파악**
- `kubectl get pod -o wide` → 상태 / 노드 배정 여부 / 재시작 횟수 확인
    - `Pending` → 스케줄링 단계에서 막힌 것
    - `CrashLoopBackOff` → 컨테이너가 뜨긴 하지만 반복 종료
    - `Init:Error` → initContainer 실패

**Step 2 - Pending이면 (스케줄링 실패 의심)**
- `kubectl describe pod <pod>` → Events에서 `FailedScheduling` 확인
    - `Insufficient cpu/memory` → 리소스 부족 → 노드 스케일 아웃 또는 requests 조정
    - `no matching node` → NodeSelector / Affinity 조건 불일치 → 스펙 확인
    - `PVC not bound` → PVC가 Pending 상태 → StorageClass / PV 확인
        - `kubectl describe pvc <pvc>` → provisioner 오류 여부 확인

**Step 3 - CrashLoopBackOff이면 (컨테이너 실행 실패 의심)**
- `kubectl logs <pod> --previous` → 크래시 직전 앱 로그 확인
- `kubectl describe pod <pod>` → 종료 코드 확인
    - Exit Code 1: 앱 내부 오류 → 로그에서 원인 파악
    - Exit Code 137: OOMKilled → `resources.limits.memory` 상향 조정
    - Exit Code 143: SIGTERM → livenessProbe 실패로 kubelet이 강제 종료
- 환경변수 / ConfigMap / Secret 누락 여부 확인
- livenessProbe `initialDelaySeconds` 너무 짧은지 검토

**Step 4 - kubectl도 이상하거나 노드 레벨 문제 의심**
- `kubectl get node` → 해당 노드 `NotReady` 여부 확인
- 노드 접속 후 kubelet 상태 확인
    - `systemctl status kubelet`
    - `journalctl -u kubelet --since "10 minutes ago"`
- container runtime 확인
    - `journalctl -u containerd -f`
