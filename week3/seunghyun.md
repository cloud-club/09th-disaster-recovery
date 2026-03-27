## 📘 스터디 주제

**Kubernetes 장애 대응을 위한 이론 정리**

## 🎯 이번 주 목표

- Kubernetes의 주요 구성요소와 역할을 이해한다.
- Pod 생성부터 트래픽 흐름까지 전체 흐름을 설명할 수 있다.
- 장애 상황 발생 시 확인해야 할 지점을 논리적으로 설명할 수 있다.

# 📚 Kubernetes 전체 아키텍처

## 1. Control Plane 구성요소 

Control Plane은 클러스터의 상태를 관리하는 컴포넌트들의 집합

### kube-apiserver

클러스터의 모든 통신이 반드시 거쳐야 하는 단일 진입점

- REST API를 노출하며, 모든 컴포넌트(kubectl, kubelet, controller, scheduler)는 이 api를 통해서 클러스터 상태를 읽거나 변경
- 유효한 요청만 etcd 기록
- 유일한 etcd 클라이언트, 다른 컴포넌트는 etcd에 직접 접근하지 않음
- 예) scheduler가 pod를 노드에 배정한 결과를 apiserver를 통해 기록

### etcd

분산 key-value 저장소로, 클러스터의 모든 상태 데이터를 저장

- 모든 k8s 오브젝트(pod, service, configmap, secret, namespace 등)의 스펙과 상태, 클러스터 설정 등을 저장
- kube-apiserver로부터 읽기/쓰기 요청을 받음
- watch 기능 제공, 특정 key 변경 시 apiserver에 이벤트 전달
- etcd가 유실되면 클러스터의 모든 상태 정보가 사라짐, etcd 백업은 필수

### kube-scheduler

노드가 배정되지 않은 Pod를 감지하여, 적절한 노드에 배정

- pod를 직접 실행하지 않음. 배정 정보를 apiserver에 기록함
  - 실제 컨테이너 실행은 해당 노드의 kubelet이 담당
- 동작 방식
  - Filtering: 모든 노드 중 해당 pod를 실행할 수 없는 노드 제외
  - Scoring: 통과한 노드에 점수를 매겨 적합한 노드 선택

### kube-controller-manager

Desired State와 Current State의 차이를 감지하고 해소하는 여러 Controller들을 단일 프로세스 안에서 병렬로 실행하는 컴포넌트

- ReplicaSet, Node, Job 등 각 Controller는 담당 리소스에 대해 Desired State와 Current State를 비교
  - 다르다면 Current State를 Desired State에 맞추는 작업을 수행
- 각 Controller를 개별 프로세스로 분리하지 않고, kube-controller-manager라는 단일 프로세스 안에서 병렬로 실행



## 2. Node 구성요소

Worker Node는 실제로 컨테이너가 실행되는 머신, 3개의 컴포넌트로 구성

### kubelet

각 Node에서 실행되는 Agent Process, 해당 Node에서 Pod의 생명주기를 관리

- Node의 OS 위에서 시스템 프로세스(systemd 서비스)로 직접 실행된다

### container runtime

컨테이너의 실제 생성/실행/중지/삭제를 담당

- kubelet이 CRI(container runtime interface)를 통해 요청을 보냄
  - 요청을 받아 컨테이너 이미지 Pull 및 생성
  - Linux 커널의 namespace/cgroup 설정하여 컨테이너 실행
- 주요 런타임
  - containerd
  - CRI-O

### kube-proxy

각 Node에서 실행되며, Service 오브젝트에 정의된 네트워크 규칙을 Node 레벨에서 구현하는 컴포넌트

- 커널의 iptables/IPVS 규칙을 설정하는 역할
- 실제 패킷 포워딩은 Linux 커널이 수행



## 3. Pod 생성 요청 시 내부 동작 흐름

1. **kubectl** → apiserver에 Pod 생성 요청 전송
2. **apiserver** → 인증 → 인가 → Admission Control 통과 → etcd에 Pod 오브젝트 저장 (nodeName 비어있음)
3. **Scheduler** → nodeName이 빈 Pod 감지 → Filtering → Scoring → 선택된 노드로 binding 정보를 apiserver에 기록
4. **kubelet** → 자기 노드에 배정된 Pod 감지 → Volume 마운트, Secret/ConfigMap 준비 → 컨테이너 런타임에 CRI 호출
5. **Container Runtime** → 이미지 Pull → namespace/cgroup 설정 → 컨테이너 실행
6. **kubelet** → Pod 상태를 Running으로 apiserver에 보고
7. **kube-proxy** → Service에 매칭되는 경우 iptables/IPVS 규칙 갱신

모든 컴포넌트는 apiserver를 통해서만 통신하며, 컴포넌트 간 직접 통신은 없음

# 📚 Pod Lifecycle & 상태

## 1. Pod 상태

### Pending

- Pod가 etcd에 생성(오브젝트로서의 Pod정보)되었지만 아직 컨테이너가 실행되지 않은 상태
- 노드 배정 대기 , 이미지 Pull , Volume 마운트 대기 등에 해당

### Running

- Pod 내 컨테이너가 정상적으로 실행 중인 상태

### CrashLoopBackOff

- 컨테이너가 시작 직후 반복적으로 실패
- kubelet이 재시작 간격을 점점 늘려가며 대기하는 상태

### Terminating

- Pod 삭제 요청 후 종료를 기다리는 상태
- 시간안에 종료되지 않은 컨테이너는 SIGKILL로 강제 종료
- 모든 컨테이너가 종료되면 Pod 오브젝트 etcd에서 삭제



## 2. Probe

### livenessProbe

- 컨테이너가 정상 동작 중인지 주기적으로 확인
- 실패 시 kubelet이 컨테이너 재시작

### readinessProbe

- 컨테이너가 트래픽을 받을 준비가 되었는지 확인
- 실패 시 해당 Pod의 IP를 Endpoint에서 제거
  - 트래픽이 해당 Pod로 전달 x



## 3. CrashLoopBackOff 발생 원인

- 애플리케이션 코드 에러로 프로세스가 즉시 종료
- 필요한 환경변수, ConfigMap, Secret이 누락
- 이미지 내 실행 파일 경로 잘못 지정
- 의존 서비스(DB 등)에 연결 실패하여 프로세스가 종료



## 4. 서비스는 정상인데 트래픽이 안 붙는 경우

- readinessProbe가 실패하고 있어 Endpoints에서 Pod IP가 제거된 상태
- Service의 selector와 Pod의 label이 불일치
- Pod의 containerPort와 Service의 targetPort가 불일치

# 📚 Kubernetes Networking

## 1. Service 종류

### ClusterIP

- 클러스터 내부에서만 유효한 가상 IP, 실제 네트워크 대상이 아님
- iptables 규칙에 의해 실제 Pod IP로 변환되어 트래픽을 전달
- Service의 기본 타입

### NodePort

- ClusterIP를 자동 생성하고, 추가로 모든 노드의 특정 포트를 열어 외부 트래픽 수신
- 외부 → 노드IP:NodePort → ClusterIP → Pod로 전달

### LoadBalancer

- NodePort를 자동 생성하고, 추가로 클라우드의 로드밸런서를 프로비저닝하여 외부에 단일 엔드포인트를 제공



## 2. Pod → Service → Pod 트래픽 흐름

- Pod-A가 Service의 ClusterIP(예: 10.96.0.10:80)로 요청 전송
- 해당 노드의 커널이 iptables 규칙을 참조하여 목적지 IP를 백엔드 Pod 중 하나의 실제 IP(예: 10.244.2.3:8080)로 DNAT 변환
- 패킷이 변환된 Pod IP로 전달되어 Pod-B가 요청을 수신

# 장애 시나리오

## 특정 노드에서만 Service 통신 실패 

### 1. 정상 흐름

Service를 통해 트래픽이 Pod까지 전달되는 과정

```
Deployment 적용 → Pod가 각 노드에 배치됨
    ↓
Service 생성 → ClusterIP 할당 → Endpoints에 Pod IP 등록
    ↓
각 노드의 kube-proxy가 apiserver를 watch하여 변경 감지
    ↓
모든 노드의 iptables 규칙 갱신 (ClusterIP → Pod IP DNAT 규칙)
    ↓
어떤 노드의 Pod에서든 ClusterIP로 요청하면
해당 노드 커널의 iptables가 실제 Pod IP로 변환하여 전달
```

핵심은 kube-proxy가 노드 단위로 동작한다는 것이다. 각 노드마다 kube-proxy가 따로 있고, 각자 자기 노드의 iptables 규칙을 관리한다.



### 2. 장애 상황

3개 노드 클러스터에서 Node-A의 Pod에서만 Service 통신이 실패한다.
Node-B, C의 Pod에서는 같은 Service로 정상 통신된다.

```
Node-A의 Pod → ClusterIP로 요청 → iptables 규칙 없음 → 전달 실패
Node-B의 Pod → ClusterIP로 요청 → iptables 정상 → Pod에 도달
Node-C의 Pod → ClusterIP로 요청 → iptables 정상 → Pod에 도달
```

Node-A의 kube-proxy가 비정상 종료되면서 iptables 규칙이 갱신되지 않는 상태다. 새로 생성된 Service나 Endpoints 변경이 Node-A의 iptables에 반영되지 않아, Node-A에 위치한 Pod가 ClusterIP로 요청을 보내도 DNAT 규칙이 없어 트래픽이 전달되지 않는다.



### 3. 장애 대응 방법

1단계 — 문제가 특정 노드에서만 발생하는지 확인

```
kubectl get pods -o wide
```

실패하는 Pod가 어떤 노드에 위치하는지 특정한다. 특정 노드의 Pod에서만 실패하면 해당 노드의 문제로 범위를 좁힌다.

2단계 — 해당 노드의 kube-proxy 상태 확인

```
kubectl get pods -n kube-system -o wide | grep kube-proxy
```

해당 노드의 kube-proxy Pod가 Running인지, CrashLoopBackOff인지 확인한다.

3단계 — kube-proxy 로그 확인

```
kubectl logs <kube-proxy-pod> -n kube-system
```

에러 메시지로 원인을 파악한다.

4단계 — 해당 노드의 iptables 규칙 확인

```
# 해당 노드에 접속하여
iptables -t nat -L | grep <ClusterIP>
```

정상 노드와 비교해서 규칙이 누락되었는지 확인한다.

5단계 — 조치

```
# kube-proxy는 DaemonSet이므로 Pod 삭제하면 자동 재생성
kubectl delete pod <kube-proxy-pod> -n kube-system
```

재시작 후 kube-proxy가 apiserver로부터 현재 Service/Endpoints 정보를 다시 받아 iptables 규칙을 복구한다.

