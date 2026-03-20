# 기초 개념 정리

## 1. SLI / SLO / SLA란?

- SLI (Service Level Indicator): 서비스 상태를 수치로 측정한 지표입니다. 예를 들어 응답 시간, 에러율, 가용성 같은 값이 SLI가 됩니다.
- SLO (Service Level Objective): 서비스가 내부적으로 목표로 삼는 운영 기준입니다. 예: 월간 가용성 99.9% 유지, 응답 시간 300ms 이하.
- SLA (Service Level Agreement): 서비스 제공자와 사용자 사이의 공식적인 약속입니다. 보통 목표를 지키지 못했을 때의 책임, 보상, 대응 기준까지 포함합니다.

즉, SLI는 "무엇을 측정할 것인가", SLO는 "어느 수준까지 목표로 할 것인가", SLA는 "그 목표를 계약이나 약속 수준으로 어떻게 보장할 것인가"에 가깝습니다.

## 2. Metrics / Logs / Traces란?

- Metrics: CPU, 메모리, 요청 수, 에러율, 지연 시간처럼 시스템 상태를 숫자로 빠르게 보는 데이터입니다. 전체 상태를 한눈에 파악하거나 알람을 걸 때 주로 사용합니다.
- Logs: 특정 시점에 어떤 일이 있었는지 남기는 이벤트성 기록입니다. 에러 메시지, stack trace, 사용자 요청 정보처럼 장애 원인을 자세히 확인할 때 유용합니다.
- Traces: 하나의 요청이 여러 서비스와 DB를 거치면서 어떻게 처리됐는지 흐름을 따라가는 데이터입니다. 마이크로서비스 환경에서 병목이나 지연 구간을 찾을 때 특히 유용합니다.

많이 사용되는 도구 예시는 다음과 같습니다.

- Metrics: Prometheus, Grafana, Datadog, New Relic, Grafana Cloud
- Logs: ELK(Elasticsearch, Logstash, Kibana), Loki, Splunk, Datadog Logs
- Traces: Jaeger, Zipkin, Grafana Tempo, Datadog APM, New Relic Distributed Tracing

## 장애를 어떻게 대응해야 할까?

개인적으로 클라우드 환경의 장애 대응은 단순히 명령어를 많이 아는 것보다, **문제가 어느 계층에서 발생했는지 추론하는 과정**이 더 중요하다고 생각합니다.

클라우드 환경은 서비스, 플랫폼, 인프라, 스토리지, 네트워크가 서로 얽혀 있기 때문에 겉으로 보이는 증상만 보고 판단하면 원인을 놓치기 쉽습니다. 그래서 장애가 발생하면 먼저 **어떤 레이어의 문제인지 가설을 세우고**, 서비스에서 보이는 현상을 플랫폼과 인프라 레벨까지 연결해서 확인해야 합니다.

또한 반복적으로 발생할 수 있는 점검 과정은 최대한 **자동화**해야 한다고 생각합니다. 사람이 매번 수동으로 확인하는 방식은 대응 속도와 정확도에 한계가 있기 때문에, 좋은 장애 대응은 결국 관측, 알람, 점검 절차가 자동화된 시스템으로 이어져야 한다고 봅니다.

## 장애 대응에서 가장 중요한 것은 무엇일까?

가장 중요한 것은 **장애를 두려워하지 않고 원인을 끝까지 추적하는 태도**라고 생각합니다.

클라우드 환경에서는 내 실수 때문이 아니더라도 장애가 언제든 발생할 수 있습니다. 그래서 장애 자체를 숨기거나 피하는 것보다, 왜 이런 일이 발생했는지 분석하고 같은 문제가 다시 생기지 않도록 **재발 방지 체계**를 만드는 것이 더 중요합니다.

결국 좋은 엔지니어는 장애를 단순히 "빨리 끄는 사람"이 아니라, 장애를 통해 시스템을 더 잘 이해하고 더 안전한 구조로 개선하는 사람이라고 생각합니다.


# 장애 대응 플로우 런북

## 1. 장애 시나리오

학생들이 사용하는 **Kubernetes 기반 JupyterHub 환경**에서 Kubernetes API Server가 정상적으로 동작하지 않는 장애가 발생한 상황입니다.

### 환경

- GPU 클러스터
- 약 **6개 노드 규모 Kubernetes 클러스터**
- 사용자 서비스: **JupyterHub**
- 스토리지: **Ceph 기반 스토리지 (PV/PVC 사용)**

### 장애 현상

- Kubernetes **API Server Pod가 CrashLoopBackOff 상태 반복**
- `kubectl` 명령 응답 지연 또는 실패
- 사용자 입장에서는
  - JupyterHub 로그인 실패
  - Notebook 생성 실패

---

## 2. 장애 인지

### 인지 방법

- 학생 사용자들의 **JupyterHub 접속 불가 문의**
- Kubernetes control plane 상태 점검 과정에서 이상 확인

### 확인된 증상

```
kubectl get pods -n kube-system
```

결과

```
E0805 23:40:29.871155 1446922 memcache.go:265] couldn't get current server API group list: Get "https://192.168.3.225:6443/api?timeout=32s": dial tcp 192.168.3.225:6443: connect: connection refused
E0805 23:40:29.871403 1446922 memcache.go:265] couldn't get current server API group list: Get "https://192.168.3.225:6443/api?timeout=32s": dial tcp 192.168.3.225:6443: connect: connection refused
E0805 23:40:29.872995 1446922 memcache.go:265] couldn't get current server API group list: Get "https://192.168.3.225:6443/api?timeout=32s": dial tcp 192.168.3.225:6443: connect: connection refused
E0805 23:40:29.874500 1446922 memcache.go:265] couldn't get current server API group list: Get "https://192.168.3.225:6443/api?timeout=32s": dial tcp 192.168.3.225:6443: connect: connection refused
E0805 23:40:29.875994 1446922 memcache.go:265] couldn't get current server API group list: Get "https://192.168.3.225:6443/api?timeout=32s": dial tcp 192.168.3.225:6443: connect: connection refused
```

추가 확인

```
journalctl -u kubelet
```

- Kubelet 로그에서 ETCD와의 통신 문제가 있는 것을 발견

---

## 3. 대응 플로우

장애 인지 후 다음 순서로 원인을 분석했습니다.

```
1. kubelet / Control Plane 이상 여부 1차 확인
   journalctl -u kubelet -n 200 --no-pager
   crictl ps -a | egrep 'kube-apiserver|etcd|kube-controller-manager|kube-scheduler'

2. API Server 컨테이너 로그 확인
   crictl ps -a | grep kube-apiserver
   crictl logs <kube-apiserver-container-id>

3. ETCD 상태 확인
   crictl ps -a | grep etcd
   crictl logs <etcd-container-id>
   ETCDCTL_API=3 etcdctl endpoint health --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
     --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

4. 살아있는 Ceph 컨테이너로 진입해 스토리지 상태 확인
   docker ps | grep ceph
   docker exec -it <ceph-container-name> bash
   ceph -s
   ceph health detail
   ceph df

5. 사용량이 큰 OSD 확인 및 불필요한 OSD 정리
   ceph osd df
   ceph osd tree
   ceph osd safe-to-destroy <osd-id>
   ceph osd out <osd-id>
   ceph osd purge <osd-id> --yes-i-really-mean-it

6. 노드 디스크 / 마운트 사용량 확인
   df -h
   lsblk

7. Ceph 사용량 임계치 확인
   ceph osd df
   ceph osd dump | grep -E 'full_ratio|backfillfull_ratio|nearfull_ratio'

8. API Server 복구 후 cluster 상태 확인
   kubectl get pods -n kube-system
   kubectl get nodes

9. Pod 이벤트 및 상세 상태 확인
   kubectl get pods -n kube-system | grep kube-apiserver
   kubectl describe pod -n kube-system <kube-apiserver-pod-name>

10. Persistent Volume / PVC 상태 확인
   kubectl get pv
   kubectl get pvc -A
   kubectl describe pvc <pvc-name> -n <namespace>
```

### 원인 발견 과정

초기에는 API Server 로그에서 명확한 오류가 보이지 않아 원인을 파악하기 어려웠습니다.

팀원과 함께 가능한 시나리오를 검토했습니다.

- Control plane 리소스 부족
- etcd 문제
- PV/PVC mount 문제
- Ceph 스토리지 문제

API Server가 복구되지 않아 `kubectl` 기반 확인이 어려웠기 때문에, 살아있는 Ceph 컨테이너에 직접 들어가 스토리지 상태를 확인했습니다.

Ceph 상태 확인 결과

```
ceph df
```

Ceph 스토리지 사용량이 **임계치를 초과하여 nearfull / full 상태**에 도달한 것을 확인했고, 사용하지 않는 대용량 OSD를 정리한 뒤 API Server가 다시 동작하기 시작했습니다.

---

## 4. 초기 대응

서비스 복구를 위해 다음 조치를 수행했습니다.

### 1️⃣ Ceph 스토리지 용량 확보

- 살아있는 Ceph 컨테이너에 접속
- `ceph osd df`로 사용량이 큰 OSD 확인
- 사용하지 않는 대용량 OSD 정리로 즉시 공간 확보

### 2️⃣ 스토리지 증설

- Ceph OSD 디스크 추가
- 스토리지 capacity 확장

### 3️⃣ Kubernetes 컴포넌트 정상화 확인

```
kubectl get pods -n kube-system
```

결과

```
kube-apiserver   Running
```

### 4️⃣ 사용자 서비스 확인

- JupyterHub 접속 정상화
- Notebook 생성 테스트

---

## 5. 원인 분석

### Root Cause

**Ceph 스토리지 용량 부족으로 인한 노드 디스크 압박 및 ETCD write 실패**

당시에는 우선 서비스 복구를 위해 저장공간 확보에 집중했고, API Server가 왜 내려갔는지까지 깊게 확인하지는 못했습니다.

지금 다시 원인을 정리해보면, Ceph 스토리지 사용량이 임계치를 초과하면서 노드의 물리 디스크 여유 공간도 함께 부족해졌고, 그 영향으로 ETCD가 정상적으로 write 하지 못했을 가능성이 높다고 보고 있습니다.

그 결과 다음과 같은 흐름으로 장애가 발생한 것으로 추정하고 있습니다.

- Ceph 스토리지 사용량 임계치 초과
- 노드 물리 디스크 여유 공간 부족
- ETCD write 실패 또는 지연
- Control plane 컴포넌트 동작 이상
- 결과적으로 **API Server CrashLoopBackOff 발생**

### 왜 문제가 발생했는가?

- 사용자 Notebook 데이터 증가
- 스토리지 사용량 모니터링 부족
- Ceph capacity alert 부재

---

## 6. 사후 방지 대책 🚀

장애 이후 단순히 디스크 증설로 끝내지 않고 **운영 환경 개선 작업을 진행했습니다.**

### 1️⃣ 스토리지 모니터링 체계 구축

Ceph 스토리지 장애를 사전에 감지하기 위해 **3계층 스토리지 모니터링 체계**를 구축했습니다.

모니터링은 **Prometheus + Grafana** 기반으로 구성했습니다.

#### Ceph Cluster Level

- Ceph cluster usage
- OSD usage
- nearfull / full 상태

#### Kubernetes Storage Level

- PV / PVC 상태
- PVC Pending 상태
- StorageClass 상태

#### Node Level

- Node 물리 디스크 사용량
- filesystem usage

대시보드를 통해 다음 지표를 통합적으로 확인할 수 있도록 구성했습니다.

- Ceph cluster usage
- Ceph OSD usage
- PVC usage
- Node disk usage
- Storage capacity trend

---

### 2️⃣ 임계치 기반 Alert 설정

스토리지 용량 문제를 사전에 감지하기 위해 Alert 정책을 설정했습니다.

예시

- Ceph usage **70% → Warning**
- Ceph usage **85% → Critical**
- Node disk usage **80% → Alert**

Alert 전달 방식

- Discord Webhook 알림 (카카오톡이나 slack같은 다른 플랫폼을 쓰고싶었는데 디스코드 봇에 비해 너무 복잡했습니다.. ㅠㅠ)

---

### 3️⃣ 운영 가이드 문서화

다음 내용을 포함한 **스토리지 운영 가이드 문서 작성**

- Ceph capacity 관리 방법
- PV/PVC 모니터링 방법
- 스토리지 증설 절차

---

## 정리

이 장애 대응 경험을 통해 **안정적인 시스템 운영은 단순히 장애를 해결하는 것이 아니라 장애의 근본 원인을 이해하고 재발 방지 체계를 만드는 과정이라는 것을 배울 수 있었습니다.**

특히 스토리지와 같은 **인프라 자원의 임계치 기반 모니터링 체계 구축이 안정적인 서비스 운영에 매우 중요하다는 것을 체감한 경험이었습니다.**