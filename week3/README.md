# 📘 Week 3 - Kubernetes 장애 대응을 위한 이론 정리

> 목표: 장애 대응 실습 전, Kubernetes의 전체 흐름과 구조를 이해하고  
> “문제가 발생했을 때 어디부터 봐야 하는지”에 대한 기준을 세운다.

---

## 🎯 이번 주 목표

- Kubernetes의 주요 구성요소와 역할을 이해한다.
- Pod 생성부터 트래픽 흐름까지 전체 흐름을 설명할 수 있다.
- 장애 상황 발생 시 확인해야 할 지점을 논리적으로 설명할 수 있다.

---

## 📚 학습 주제

### 1️⃣ Kubernetes 전체 아키텍처

- Control Plane 구성요소
  - kube-apiserver
  - etcd
  - scheduler
  - controller-manager

- Node 구성요소
  - kubelet
  - container runtime
  - kube-proxy

📌 학습 포인트
- Pod 생성 요청 시 내부 동작 흐름 이해
- 각 컴포넌트의 역할과 책임 구분

---

### 2️⃣ Pod Lifecycle & 상태

- Pod 상태
  - Pending
  - Running
  - CrashLoopBackOff
  - Terminating

- Probe
  - livenessProbe
  - readinessProbe

📌 학습 포인트
- CrashLoopBackOff가 발생하는 이유
- 서비스는 정상인데 트래픽이 안 붙는 경우의 원인

---

### 3️⃣ Kubernetes Networking

- Service 종류
  - ClusterIP
  - NodePort
  - LoadBalancer

- kube-proxy 역할 (iptables 기반)
- Pod → Service → Pod 트래픽 흐름

📌 학습 포인트
- Service를 통해 트래픽이 전달되는 과정
- 특정 노드에서만 통신이 안되는 이유

---

### 4️⃣ Kubernetes Storage

- PV / PVC 개념
- StorageClass
- Volume attach 흐름

📌 학습 포인트
- PVC Pending 상태의 원인
- 스토리지 문제와 Pod 생성 실패의 관계

---

### 5️⃣ Workload 리소스

- Deployment / ReplicaSet
- Desired State 개념
- Rolling Update 동작 방식

📌 학습 포인트
- Pod가 계속 재생성되는 이유
- 수동으로 삭제한 Pod가 다시 생성되는 이유

---

### 6️⃣ 기본 디버깅 방법

- kubectl logs
- kubectl describe
- kubectl get events

📌 학습 포인트
- 이벤트 기반 문제 추적 방법
- kubectl이 정상 동작하지 않을 때 확인해야 할 위치
  - kubelet 로그
  - container runtime 로그
  - journalctl

---

## 🧠 과제 (발표 준비)

각자 맡은 주제에 대해 아래 내용을 포함하여 정리 및 발표합니다.

### ✔ 1. 정상 흐름 설명

- 해당 영역에서의 정상 동작 흐름을 설명
- (예: Pod 생성 → 스케줄링 → 실행 → 서비스 연결)

---

### ✔ 2. 장애 상황 정의

- 해당 영역에서 발생할 수 있는 장애 상황 1가지 정의

예시:
- API Server 장애
- kubelet 장애
- PVC Pending 상태
- Service 라우팅 실패

---

### ✔ 3. 장애 대응 방법

- 문제 발생 시 확인해야 할 순서 정리

예시:
- 어떤 리소스를 먼저 확인할 것인가
- 어떤 로그를 확인할 것인가
- 어떤 컴포넌트를 의심할 것인가

---

## 📌 제출 방식

- 본인 이름 브랜치 생성
- `week3/{본인이름}.md` 또는 개별 정리 파일 업로드
- PR 생성

---

## 🔥 중요

> 이번 주는 "개념 암기"가 아니라  
> "장애 발생 시 어디부터 확인할 것인가"를 기준으로 학습합니다.

단순 정의 나열이 아닌  
👉 흐름 + 장애 발생 지점 + 대응 방법  
까지 연결해서 정리해주세요.

---

## 🚀 다음 주 예고

- 실제 장애 상황을 기반으로 한 실습 진행
- 클러스터 내 다양한 장애 시나리오 대응

👉 이번 주 내용을 제대로 이해해야 다음 주 실습이 의미 있습니다.
