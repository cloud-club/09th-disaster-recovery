---

**✔ 1. 정상 흐름 설명**

- 해당 영역에서의 정상 동작 흐름을 설명
- (예: Pod 생성 → 스케줄링 → 실행 → 서비스 연결)

---

**✔ 2. 장애 상황 정의**

- 해당 영역에서 발생할 수 있는 장애 상황 1가지 정의

예시:

- API Server 장애
- kubelet 장애
- PVC Pending 상태
- Service 라우팅 실패

---

**✔ 3. 장애 대응 방법**

- 문제 발생 시 확인해야 할 순서 정리

예시:

- 어떤 리소스를 먼저 확인할 것인가
- 어떤 로그를 확인할 것인가
- 어떤 컴포넌트를 의심할 것인가

---

## 🧯 장애 대응 플로우 런북

---

### 1️⃣ 장애 시나리오

Spring Boot 기반 애플리케이션이 Kubernetes 환경에서 배포된 이후 Pod가 정상적으로 기동되지 않고 지속적으로 재시작되며 CrashLoopBackOff 상태에 빠지는 장애가 발생.

해당 서비스는 Spring Boot Actuator를 활용하여 health check endpoint를 구성하고 있었으며 Kubernetes의 livenessProbe와 readinessProbe를 통해 상태를 확인하고 있었다.

---

### 2️⃣ 장애 인지

장애는 ArgoCD 대시보드를 통해 인지

- Pod 재시작 횟수 증가 (restart count 급증)
- 서비스 요청 실패율 증가
- 일부 API 응답 지연 발생

또한 Kubernetes 상태 확인 시 다음과 같은 현상을 확인하였다.

- `kubectl get pods` 결과 CrashLoopBackOff 상태 확인
- readinessProbe 및 livenessProbe 실패 이벤트 발생

---

### 3️⃣ 대응 플로우

장애 인지 이후 다음 순서로 점검을 진행

<aside>
⚠️

kubelet이 livenessProbe 실패 시 컨테이너를 재시작하는 구조이므로 kubelet의 probe 판단 로직을 중심으로 문제를 분석

</aside>

1. **Pod 상태 확인**
    
    ```bash
    kubectl get pods
    ```
    
    → CrashLoopBackOff 상태 확인
    
2. **이벤트 확인**
    
    ```bash
    kubectl describe pod <pod-name>
    ```
    
    → livenessProbe 실패 이벤트 확인
    
3. **애플리케이션 로그 확인**
    
    ```bash
    kubectl logs <pod-name>
    ```
    
    → 애플리케이션 자체 오류는 없고 정상 기동 중인 것으로 판단
    
4. **Probe 설정 확인**
    
    → livenessProbe 및 readinessProbe의 설정값 점검
    
5. **헬스체크 endpoint 확인**
    
    ```bash
    curl localhost:8081/actuator/health/liveness
    ```
    
    → 초기 기동 시점에는 응답이 지연되는 것을 확인
    
6. **원인 가설 수립**
    
    → 애플리케이션 기동 시간보다 probe 실행 시점이 더 빠른 것으로 판단
    

---

### 4️⃣ 초기 대응

서비스 영향을 최소화하기 위해 다음과 같은 조치를 수행

- livenessProbe의 initialDelaySeconds 값을 증가시켜
    
    애플리케이션 기동 전에 probe가 실행되지 않도록 조정
    
- readinessProbe의 delay를 증가시켜
    
    준비되지 않은 Pod가 트래픽을 받지 않도록 설정
    

이를 통해 Pod의 반복 재시작을 방지하고 서비스 안정성을 확보

---

### 5️⃣ 원인 분석

애플리케이션 자체에는 문제가 없었으나 Kubernetes의 livenessProbe가 애플리케이션 기동 완료 이전에 실행되면서 정상적인 컨테이너를 비정상 상태로 오인한 것이 원인. 이로 인해 kubelet이 컨테이너를 반복적으로 재시작하였고 결과적으로 CrashLoopBackOff 상태가 발생

---

### 6️⃣ 사후 방지 대책 🚀

동일한 문제가 재발하지 않도록 다음과 같은 개선을 적용함

```bash
startupProbe: 
	httpGet: 
		path: /actuator/health/liveness 
		port: 8081 
	failureThreshold: 40 
	periodSeconds: 5
```

1. **startupProbe 도입**
    - 애플리케이션 기동 완료 전까지 livenessProbe와 readinessProbe 실행을 지연
    - 기동 시간과 헬스체크 시점을 명확히 분리
2. **Probe 역할 분리 명확화**
    - livenessProbe: 컨테이너 재시작 기준
    - readinessProbe: 트래픽 수신 가능 여부 판단
3. **헬스체크 endpoint 개선**
    - `/actuator/health/liveness`와 `/actuator/health/readiness` 분리
    - 상태별 응답 코드 명확화
4. **모니터링 강화**
    - Pod restart count 기반 alert 설정
    - probe 실패 이벤트 모니터링 추가
5. **설정 표준화**
    - 서비스별 공통 probe 설정 가이드 정의
    - startupProbe 사용을 기본 정책으로 적용

---

### ✅ 결론

본 장애는 애플리케이션 문제가 아닌 **Kubernetes Probe 설정과 애플리케이션 기동 시간 간의 불일치**로 인해 발생하였다.

이를 통해 Pod Lifecycle과 kubelet의 동작 방식을 이해하고 적절한 probe 전략을 수립하는 것이 안정적인 서비스 운영에 중요하다는 것을 확인하였다.
