### 장애/에러 탐지를 위한 좋은 알림의 조건은?

1.  **중요한 것만 울리도록 한다.** 알림을 설정하다보면 알림의 범위를 고르는 순간이 오는데 장애 / 에러 탐지의 알림은 아주 중요하지만, 알림이 너무 많이 오게 되면 결국 무시하게 된다. 그렇기 때문에 중요한 것만 울리도록 하는 것이 중요하다.

2. 알림을 받았을 때, 단순히 **에러 알림을 받는 것**이 아니라, 장애 / 에러에 바로 대응이 가능해야 한다. 

    - 어떤 서비스 / 리소스에서 발생했는지가 명확하고
    - 원인 추정 정보를 포함하는 알림이어야 한다
    - 또한, 대응 가능하도록 슬랙과 같은 경우는 담당자 태그 요런 사소한 것도 포함

3. **당연하게도 알림을 신뢰할 수 있어야 한다.** 알림이 틀리면 아무도 믿지 않게되는 양치기 소년이 된다… 그렇기 때문에 테스트 환경에서 충분하게 검증을 한다.

---

### 사용해본 Alert 툴 & 좋았던/별로였던 점

- **Grafana**
    - 좋았던 점
        - 시각화 + 알림 연결이 자연스러움 (통합해서 파악이 쉬움)
        - Prometheus, Loki 등 연동 편리
    - 별로였던 점
        - alert 복잡도 증가 시 관리 어려움
        - 직접 튜닝이 많이 필요함

- **Robusta**
    - 좋았던 점
        - kubernetes 친화적, k8s 이벤트 기반으로 자동 대응 가능
        - 무료 오픈소스임 ㅎㅎ
        - 자동 Runbook 액션 제공
        - slack 연동이 쉽고 강력하다 (알림 메시지 자체 풍부)
        
        ```bash
        # 실제로 받은 robusta 알림
        :eyes: K8s event detected :red_circle: High
        Crashing pod prometheus-server-server in namespace monitoringSource: nks-clusterCrash Info 
        ● Container prometheus-server
        ● Restarts 2
        ● Status WAITING
        ● Reason CrashLoopBackOffPrevious Container 
        ● Status TERMINATED
        ● Reason Error
        ● Started at 2026-03-04T07:07:37Z
        ● Finished at 2026-03-04T07:07:37ZAsk AI questions about this alert, by adding holmes to your Slack.
        
        # 아래에는 런북
        ```