**1️⃣ Kubernetes 전체 아키텍처**

- Control Plane 구성요소
    - kube-apiserver
        - 모든 요청을 처리하는 클러스터의 관문
    - etcd
        - 클러스터 상태를 저장하는 데이터베이스
    - scheduler
        - Pod를 실행할 노드를 결정
    - controller-manager
        - 클러스터를 원하는 상태로 유지하는 관리자
- Node 구성요소
    - kubelet
        - 노드에서 Pod를 실행·관리하는 에이전트
    - container runtime
        - 컨테이너를 실제로 실행하는 엔진
    - kube-proxy
        - 서비스 트래픽을 Pod로 전달하는 네트워크 관리자

---

**2️⃣ Pod Lifecycle & 상태**

- Pod 상태
    - Pending
        - Pod가 생성되었지만 아직 Node에 배치되거나 실행 준비가 안 된 상태
    - Running
        - Pod가 정상적으로 실행 중인 상태
    - CrashLoopBackOff
        - 컨테이너가 계속 실패하며 재시작 반복 중인 상태
    - Terminating
        - Pod가 종료되는 중인 상태
- Probe
    - livenessProbe
        - 컨테이너가 살아있는지 확인 (문제 시 재시작)
    - readinessProbe
        - 트래픽을 받을 준비가 되었는지 확인 (준비 안 되면 서비스에서 제외)

---

**3️⃣ Kubernetes Networking**

- Service 종류
    - ClusterIP
        - 클러스터 내부에서만 접근 가능한 기본 서비
    - NodePort
        - 각 노드의 포트를 통해 외부에서 접근 가능한 서비스
    - LoadBalancer
        - 외부 로드밸런서를 통해 외부에 서비스 노출
- kube-proxy 역할 (iptables 기반)
    - iptables 기반으로 서비스 트래픽을 Pod로 라우팅
- Pod → Service → Pod 트래픽 흐름
    - Service가 여러 Pod 중 하나로 트래픽을 분산 전달

---

**4️⃣ Kubernetes Storage**

- PV / PVC 개념
    - PV는 실제 스토리지 자원, PVC는 사용자가 요청하는 저장공간
- StorageClass
    - 스토리지 종류와 생성 방식을 정의하는 템플릿
- Volume attach 흐름
    - PVC 요청 → PV 생성/바인딩 → Pod에 마운트 → 컨테이너에서 사용

---

**5️⃣ Workload 리소스**

- Deployment / ReplicaSet
    - 원하는 개수의 Pod를 생성·유지하는 리소스
- Desired State 개념
    - 사용자가 정의한 상태를 기준으로 실제 상태를 지속적으로 맞추는 개념
- Rolling Update 동작 방식
    - 기존 Pod를 점진적으로 교체하며 무중단 배포 수행

---

**6️⃣ 기본 디버깅 방법**

- kubectl logs
    - Pod/컨테이너의 실행 로그를 조회
- kubectl describe
    - 리소스 상태, 이벤트, 상세 정보를 확인
- kubectl get events
    - 클러스터에서 발생한 이벤트 흐름 확인

---

# Pod Scheduling 장애 대응 매뉴얼 (Pending 상태)

## `1. 정상 흐름 설명`

- Pod는 생성 후 다음 흐름을 거쳐 실행된다.

```python
- 사용자가 Pod 또는 Deployment 생성 요청
- kube-apiserver에 리소스 등록
- scheduler가 조건을 만족하는 Node 선택
- 선택된 Node의 kubelet이 Pod 실행
- container runtime이 컨테이너 생성
- Pod가 Running 상태로 전환
```

- scheduler는 다음 요소를 기반으로 Node를 선택한다.

```python
- CPU / Memory 리소스 여유
- nodeSelector / affinity 조건
- taint & toleration
- 볼륨(PVC) 바인딩 가능 여부
```

## `2. 장애 상황 정의`

### Pod Pending (스케줄링 실패)

- Pod가 생성되었지만 Node에 배치되지 못하고 Pending 상태로 유지되는 상황

### 주요 증상

```python
- **kubectl get pods** 결과에서 STATUS가 Pending
- 일정 시간이 지나도 Running으로 전환되지 않음
- **kubectl describe pod** 에서 스케줄링 실패 이벤트 발생
```

### 예시 이벤트

```python
- 0/3 nodes available
- Insufficient cpu
- Insufficient memory
- node(s) didn't match node selector
```

### 주요 원인

```python
- 노드 리소스 부족 (CPU / Memory)
- nodeSelector 또는 affinity 조건 불일치
- taint 설정으로 인한 스케줄링 차단
- PVC 바인딩 실패 (스토리지 문제)
- Node 상태 NotReady
```

## `3. 장애 대응 방법`

### 1) Pod 상태 확인

```python
kubectl get pods
```

- Pending 상태 여부 확인

### 2) Pod 상세 정보 확인

```python
kubectl describe pod <pod-name>
```

- 이벤트(Event) 메시지 확인
- 스케줄링 실패 원인 파악

### 3) Node 상태 확인

```python
kubectl get nodes
```

- Ready 상태 여부 확인
- 노드 개수 및 상태 점검

### 4) 리소스 요청 확인

```python
kubectl describe pod <pod-name>
```

- requests / limits 값 확인
- 과도한 리소스 요청 여부 점검

### 5) 스케줄링 조건 확인

- nodeSelector 설정 확인
- affinity / anti-affinity 조건 확인
- toleration 설정 확인

### 6) 스토리지 문제 확인 (PVC 사용 시)

```python
kubectl get pvc
```

- PVC가 Pending 상태인지 확인
- 바인딩 여부 점검

## `정리`

```python
- Pending 상태에서는 describe를 통해 이벤트를 먼저 확인
- 대부분 원인은 리소스 부족 또는 스케줄링 조건 불일치
- 스케줄러 자체 문제가 아니라 설정 문제인 경우가 많음
```
