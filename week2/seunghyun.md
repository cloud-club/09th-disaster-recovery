## 🎯 스터디 주제

**서비스(종류 무관) 관측 기초 다지기**

## 🧱 기초 개념

### **SLI / SLO / SLA란?**

- SLI
  - 서비스의 상태를 판단하기 위해 실제로 수집하는 수치 데이터
  - 우리 서비스가 지금 잘 돌아가고 있는가?"라는 질문에 답하려면, 먼저 무언가를 측정해야 하는데 SLI가 바로 그 측정 항목
    - 예) 전체 HTTP 요청 중 200 OK 응답이 차지하는 비율 (가용성)
    - 예) 요청을 보낸 시점부터 응답을 받은 시점까지 걸린 시간 (지연시간)
    - 예) 초당 처리 가능한 요청 수 (처리량)

- SLO
  - SLI로 측정한 값이 어느 범위 안에 있어야 정상인지 정의한 내부 목표
  - SLI만 있으면 "응답시간이 300ms다"라는 사실은 알 수 있지만, 그게 좋은 건지 나쁜 건지 판단할 수 없음 SLO는 그 판단 기준
    - 예) 최근 30일간 전체 요청의 99.9%가 500ms 이내에 응답해야 한다
    - 예) 월간 가용성이 99.95% 이상이어야 한다

- SLA
  - SLO를 기반으로, 서비스 제공자와 고객 사이에 체결하는 공식 계약
    - 예) IT 지원 팀은 고객의 이메일 지원 요청을 1시간 이내에 확인하고 응답한다
    - 예) 월간 가용성 99.9% 미만 시, 해당 월 요금의 10%를 크레딧으로 환급한다

<br/>

### **Metrics / Logs / Traces란?**

- Metrics
  - 시스템의 상태를 일정 간격으로 수집하는 숫자 값
  - 시간 흐름에 따른 추이를 파악하는 데 적합
  - 대시보드의 그래프와 알림 설정에 주로 사용되며 SLI가 Metrics로 측정됨
    - 예) CPU 사용률 / 메모리 사용량 / 초당 HTTP 요청 수 / HTTP 에러 비율

- Logs
  - 시스템에서 발생한 개별 이벤트를 시간순으로 기록한 텍스트 데이터
  - 특정 시점에 구체적으로 무슨 일이 일어났는가?

- Traces
  - 하나의 요청이 시스템 내부의 여러 서비스를 거치는 전체 경로와 각 구간의 소요 시간을 기록한 데이터
  - 이 요청이 어떤 경로로 처리되었고, 어디서 지연이 발생했는가?
    - 예) 사용자가 주문하기 버튼을 누르면, 그 요청은 여러 서비스를 순차적/병렬적으로 거침, Tracing 시스템은 이 요청에 고유한 Trace ID를 부여하고, 각 서비스를 통과할 때마다 Span이라는 단위로 구간을 기록

<br/>
세 데이터는 독립적으로 존재하는 것이 아니라, 장애 대응 시 단계적으로 함께 사용된다. <br/>
1. Metrics로 이상 감지 /
2. Traces로 병목 구간 특정 /
3. Logs로 근본 원인 확인

## 💭 각자의 생각 (자유롭게)

### 장애를 어떻게 대응해야 할까?

먼저 장애가 발생했다는 것을 감지해야 한다고 생각합니다. 모니터링 시스템의 Alert을 통해 감지하는 것이 기본이고, 고객보다 먼저 내부에서 인지할 수 있도록 Alert 체계를 사전에 구성해두는 것이 중요하다고 생각합니다.

장애를 감지하면, 바로 해결에 들어가기 전에 이 장애의 영향 범위와 심각도를 먼저 판단하겠습니다. 전체 서비스가 중단된 상황인지, 특정 기능에 한정된 문제인지에 따라 대응의 긴급도와 투입 인원이 달라지기 때문입니다.

심각도를 판단한 후에는, 원인을 완전히 파악하기 전이라도 먼저 서비스 영향을 줄이는 데 집중하겠습니다. 예를 들어 배포 직후에 문제가 발생했다면 먼저 롤백을 수행하고, 특정 인스턴스에 문제가 있다면 해당 인스턴스에서 트래픽을 제외하는 식입니다. 만약 알려진 해결책이 없는 상황이라면, 변경 사항을 최소화하면서 가설을 하나씩 검증하는 방식으로 접근하겠습니다. 여러 조치를 동시에 적용하면 이후에 어떤 것이 효과가 있었는지 판단할 수 없기 때문입니다.

서비스가 복구된 이후에는 근본 원인 분석을 수행하겠습니다. Metrics, Logs, Traces 데이터를 활용하여 장애의 발생 원인을 특정하고, 이를 바탕으로 재발 방지 조치를 수행하겠습니다. 가능하다면 근본 원인 자체를 제거하는 것이 가장 좋고, 그것이 어렵다면 동일한 장애가 발생했을 때 빠르게 감지하고 대응할 수 있는 사전 대응 체계를 구축하겠습니다. 그리고 이 전체 과정을 문서로 남겨 팀에 공유하는 것까지가 장애 대응의 완결이라고 생각합니다.

### 장애 대응에서 가장 중요한 것은 무엇일까?

가장 중요한 것은 재발 방지라고 생각합니다.

처음 겪는 장애는 불가피하지만, 같은 원인으로 반복되는 것은 대응 체계의 부재입니다. 장애 복구 후 근본 원인을 분석하고 제거하거나, 사전 대응 체계를 구축하고, 이를 문서로 공유하여 조직 차원에서 반복을 방지해야 합니다.

다만 모든 장애를 사전에 방지할 수는 없으므로, 예측하지 못한 장애에 대해서는 MTTR을 최소화하는 것이 핵심입니다. 빠른 감지(모니터링), 빠른 판단(Runbook), 빠른 복구(롤백 자동화, 인프라 이중화)가 이를 뒷받침합니다.

정리하면, 경험한 장애는 재발 방지로 반복을 막고, 미경험 장애에 대해서는 MTTR을 최소화하는 체계를 갖추는 것이 장애 대응의 핵심이라고 생각합니다.

# 장애 대응 플로우 런북

## 1. 장애 시나리오

NAT Instance 장애 (Private Subnet 외부 통신 중단) 상황

### 환경

- 서비스: 정적 웹사이트 호스팅 플랫폼. ECS API 서버가 사용자 요청을 처리하며, 일부 기능에서 외부 API(결제, 인증 등)를 호출함.
- VPC: Public Subnet / Private Subnet 분리
- Private Subnet: ECS Fargate Task (API 서버), RDS
- Public Subnet: NAT Instance (EC2), Internet Gateway
- NAT 구성: 비용 절감을 위해 NAT Gateway 대신 NAT Instance(EC2)를 사용.
- 라우팅: Private Subnet Route Table에서 0.0.0.0/0 → NAT Instance ENI
- 모니터링: Datadog (ECS 사이드카), Alert → Slack

### 장애 현상

내부 통신은 정상인데 외부 통신만 실패

- 내부 통신(ECS ↔ RDS 등)은 Private Subnet 내에서 직접 통신
- 외부 통신(ECS → 외부 API)은 NAT Instance → Internet Gateway를 경유
- 모니터링 시스템에서 ECS API 서버의 HTTP 5xx 에러율 급증 Alert 발생
  - ECS Task 로그에서 외부 API 호출 시 Connection Timeout 에러 반복
  - ECS Task 자체는 Running 상태이며, Task 간 내부 통신은 정상
  - RDS 연결 정상
- 외부 API에 의존하는 기능(결제, 외부 인증 등)은 사용 불가.

## 2. 장애 인지

### 인지 방법

- Datadog Alert: ECS API 서버의 HTTP 5xx 에러율이 임계치 초과
- Alert 수신 채널: Slack(또는 Discord)

### 확인된 증상

- ECS Task 로그에서 외부 API 호출 시 Connection Timeout 에러 반복 확인
- ECS Task 자체는 Running 상태이며, Task 간 내부 통신(Private Subnet 내)은 정상
- RDS 연결 정상
- **패턴: 내부 통신은 정상, 외부 통신만 실패**

## 3. 대응 플로우

### 확인 순서

1. NAT Instance 상태 확인
   - AWS 콘솔 > EC2 > NAT Instance의 Instance State 확인
   - running / stopped / terminated 여부
   - Status Check (2/2 passed) 여부
     - System Status Check: AWS 물리 호스트 레벨의 문제
     - Instance Status Check: OS 레벨의 문제

2. NAT Instance의 네트워크 설정 확인
   - Source/Destination Check가 disabled인지 확인
     (NAT Instance는 자신이 출발지/목적지가 아닌 패킷을 중계하므로,
     이 설정이 enabled이면 해당 패킷을 모두 폐기함)

3. Private Subnet Route Table 확인
   - 0.0.0.0/0 → NAT Instance의 ENI 경로가 정상인지 확인
   - 상태가 blackhole로 표시되어 있는지 확인
     (NAT Instance가 terminated되면 해당 경로가 blackhole 상태로 변경됨)

4. Security Group / NACL 확인
   - NAT Instance의 Security Group
     - Inbound: Private Subnet CIDR에서 오는 트래픽 허용 여부
     - Outbound: 0.0.0.0/0 으로의 트래픽 허용 여부
   - Public Subnet의 NACL
     - Ephemeral Port(1024-65535) 허용 여부

5. NAT Instance 시스템 내부 확인 (SSH 접속이 가능한 경우)
   - IP forwarding 활성화 여부: cat /proc/sys/net/ipv4/ip_forward (값이 1이어야 함)
   - iptables NAT 규칙 확인: iptables -t nat -L
   - 디스크 사용률, 메모리 사용률, CPU 사용률 확인
   - NAT Instance 자체에서 외부 통신 테스트: curl -I https://www.google.com

## 4. 초기 대응

### Case 1: NAT Instance가 stopped 상태인 경우

- EC2 콘솔에서 해당 인스턴스를 Start
- Instance Status Check가 2/2 passed 될 때까지 대기
- 외부 통신 복구 확인

### Case 2: NAT Instance가 running이지만 Status Check 실패인 경우

- Instance Reboot 시도
- Reboot 후에도 Status Check 실패가 지속되면 → Case 3으로 전환

### Case 3: NAT Instance가 복구 불가능한 경우 (terminated, 또는 reboot 후에도 비정상)

#### 방법 A: 새 NAT Instance 생성 (근본 복구)

1. 기존 NAT Instance와 동일한 AMI로 새 EC2 인스턴스를 Public Subnet에 생성
2. Source/Destination Check 비활성화
3. EIP(Elastic IP) 할당 및 연결
4. Private Subnet Route Table의 0.0.0.0/0 Target을 새 인스턴스의 ENI로 변경
5. IP forwarding 활성화 및 iptables NAT 규칙 설정 확인
6. 외부 통신 복구 확인

#### 방법 B: NAT Gateway로 임시 전환 (가장 빠른 복구)

1. AWS 콘솔에서 Public Subnet에 NAT Gateway 생성 (생성까지 약 1~2분 소요)
2. Private Subnet Route Table의 0.0.0.0/0 Target을 NAT Gateway로 변경
3. 외부 통신 복구 확인
4. 서비스 정상화 후, 비용과 운영 방식을 고려하여 NAT Instance로 재전환할지 판단

### Case 4: NAT Instance는 정상이지만 설정 문제인 경우

- Source/Destination Check가 enabled → disabled로 변경
- Route Table이 blackhole → 정상 ENI로 Target 재지정
- Security Group / NACL 규칙 수정
- IP forwarding이 꺼져 있다면: sysctl -w net.ipv4.ip_forward=1

## 5. 원인 분석

NAT Instance 장애의 원인으로 가능한 경우들:

### 인스턴스 레벨

- EC2 물리 호스트 장애로 인한 인스턴스 중단 (AWS 측 장애)
- 인스턴스 타입의 리소스 한계 초과 (t3.micro 등 소형 인스턴스에서
  트래픽 증가 시 CPU 크레딧 소진 또는 네트워크 대역폭 초과)
- 디스크 풀(Disk Full)로 인한 OS 비정상 동작
- OOM(Out of Memory)으로 인한 프로세스 중단

### 설정 레벨

- Source/Destination Check가 의도치 않게 enabled로 변경됨
  (수동 콘솔 작업 중 실수, 또는 IaC 코드 변경 시 누락)
- iptables NAT 규칙 초기화 (인스턴스 재부팅 후 규칙이 영속화되지 않은 경우)
- ip_forward 설정 초기화 (sysctl 설정이 영속화되지 않은 경우)

### 네트워크 레벨

- Route Table 경로가 변경되거나 blackhole 상태로 전환됨
- Security Group 또는 NACL 규칙이 변경되어 트래픽 차단

## 6. 사후 방지 대책

### 단일 장애점(SPOF) 해소

- NAT Instance를 단일 인스턴스로 운영하는 것 자체가 근본적인 위험 요소
- 대책 옵션:
  - Multi-AZ에 각각 NAT Instance를 배치하고, 장애 시 Route Table을
    자동 전환하는 Health Check 스크립트 또는 Lambda 구성
  - Auto Scaling Group(min=1, max=1)으로 NAT Instance를 관리하여,
    인스턴스 장애 시 자동 재생성
  - 비용이 허용된다면 NAT Gateway로 전환 (AWS 관리형, Multi-AZ 지원)

### 모니터링 강화

- NAT Instance에 대한 전용 Alert 추가:
  - EC2 StatusCheckFailed 지표 감시
  - CPU 사용률, 네트워크 트래픽(NetworkOut) 급감 감시
  - Private Subnet에서 외부 통신 가능 여부를 주기적으로 확인하는
    Synthetic Monitoring(예: Lambda에서 외부 API를 호출하여 응답 확인)

### 설정 영속화 및 IaC 관리

- ip_forward, iptables 규칙을 /etc/sysctl.conf, iptables-save 등으로
  영속화하여 재부팅 후에도 유지되도록 설정
- NAT Instance의 모든 설정을 Terraform으로 관리하여,
  수동 변경으로 인한 설정 드리프트 방지
- Source/Destination Check 비활성화를 Terraform 코드에 명시

### Runbook 자체 개선

- 이번 장애 대응 과정을 문서로 작성하여 팀 공유
- 대응 시간 기록 (감지 → 원인 파악 → 복구 완료까지 각 단계별 소요 시간)
- 다음 장애 시 더 빠르게 대응할 수 있도록 Runbook 업데이트
