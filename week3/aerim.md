# Week 3 — Service 라우팅 실패

---

## 1. 정상 흐름

Service로 요청이 들어오면 이런 순서로 Pod까지 전달된다.

```
클라이언트가 my-service:8080 호출
    ↓
CoreDNS가 서비스 이름을 ClusterIP(가상 IP)로 바꿔줌
    ↓
kube-proxy가 iptables 규칙으로 ClusterIP → 실제 Pod IP로 변환
    ↓
readinessProbe 통과한 Pod에만 트래픽 들어옴
    ↓
Pod가 처리하고 응답 반환
```

이게 정상적으로 돌아가려면 세 가지가 맞아야 한다.

- Service의 selector랑 Pod의 label이 똑같아야 함
- Pod가 readinessProbe를 통과해야 Endpoints에 등록됨
- Service의 targetPort랑 앱이 실제로 쓰는 포트가 같아야 함

---

## 2. 장애 상황

### 시나리오: Pod는 Running인데 Service로 요청하면 아무 응답이 없음

백엔드 개발자 입장에서 제일 당황스러운 상황이다.  
Pod 자체는 살아있으니까 앱 문제인가 싶은데, 실제로는 **Endpoints가 비어있어서** 트래픽이 아예 전달이 안 되는 거다.

```
클라이언트 → Service → Endpoints 비어있음 → 보낼 Pod 없음 → 응답 없음
```

Endpoints가 비는 이유는 크게 세 가지다.

| 원인 | 상황 |
|------|------|
| label 불일치 | selector는 `app: my-app`인데 Pod label은 `app: my-app-v2` 이런 식 |
| readinessProbe 실패 | Pod는 뜨긴 했는데 DB 연결 실패나 초기화 중이라 아직 준비 안 됨 |
| 포트 번호 불일치 | targetPort는 8080인데 앱은 8081 쓰는 경우 |

---

## 3. 장애 대응 방법

### 확인 순서

#### 1단계 — Endpoints 먼저 본다

```bash
kubectl get endpoints my-service
```

비어있으면 Service가 Pod를 아예 못 찾고 있는 거다.  
IP가 있으면 포트 문제나 앱 내부 문제로 좁혀서 본다.

#### 2단계 — label이랑 selector 비교

```bash
# Service selector 확인
kubectl describe service my-service

# Pod label 확인
kubectl get pod --show-labels
```

selector랑 label 값이 완전히 일치하는지 본다. 오타 하나 차이로 Endpoints가 비는 경우가 진짜 많다.

#### 3단계 — readinessProbe 실패했나 확인

```bash
kubectl describe pod <pod-name>
# Events에 이런 메시지 있으면 readiness 실패
# Warning  Unhealthy  Readiness probe failed: ...
```

#### 4단계 — 포트 번호 확인

```bash
# Service targetPort 확인
kubectl describe service my-service

# 실제로 앱이 어느 포트 쓰는지 확인
kubectl exec -it <pod-name> -- ss -tlnp
```

#### 5단계 — 위에서 다 이상 없으면 kube-proxy 확인

```bash
kubectl get pod -n kube-system | grep kube-proxy
kubectl logs -n kube-system <kube-proxy-pod-name>
```

---

### 의심 순서 정리

```
Endpoints 비어있음
    → label 불일치 먼저 의심
    → readinessProbe 실패 확인
    → targetPort 번호 맞는지 체크

Endpoints에 IP 있는데 응답 없음
    → 앱이 실제로 쓰는 포트랑 targetPort 다른 거 아닌지 확인
    → kube-proxy 로그 확인
    → kubectl logs로 앱 에러 확인
```

---

### 한 줄 요약

> Service 라우팅 실패의 대부분은 Endpoints가 비는 것이고,  
> 원인은 거의 label 불일치 아니면 readinessProbe 실패다.  
> 뭔가 안 된다 싶으면 `kubectl get endpoints` 부터 치자.
