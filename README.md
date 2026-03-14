# 👥 참여자

<table>
  <tr>
    <td align="center">
      <a href="https://github.com/hyobin-yang">
        <img src="https://avatars.githubusercontent.com/u/101079399?v=4" width="100px;" alt="양효빈"/>
        <br />
        <sub><b>양효빈</b></sub>
      </a>
      <br />
      <sub>스터디 리더</sub>
    </td>
    <td align="center">
      <a href="https://github.com/back1ash">
        <img src="https://avatars.githubusercontent.com/u/53644611?v=4" width="100px;" alt="이해준"/>
        <br />
        <sub><b>이해준</b></sub>
      </a>
      <br />
      <sub>스터디 리더</sub>
    </td>
    <td align="center">
      <a href="https://github.com/zzzang12">
        <img src="https://avatars.githubusercontent.com/u/70265177?v=4" width="100px;" alt="이장원"/>
        <br />
        <sub><b>이장원</b></sub>
      </a>
      <br />
      <sub>스터디원</sub>
    </td>
    <td align="center">
      <a href="https://github.com/(github ID)">
        <img src="https://avatars.githubusercontent.com/u/(프로필 이미지 링크)?v=4" width="100px;" alt="(이름)"/>
        <br />
        <sub><b>(이름)</b></sub>
      </a>
      <br />
      <sub>스터디원</sub>
    </td>
  </tr>
</table>

# 스터디 방식

### 일정

- `시간`: 매주 **토요일 11시 ~ 13시**
    - 토요일 클클 행사가 있는 경우 스터디 끝나고 사이좋게 점심 먹고 행사 가요~~ㅎㅎ
- `장소`: 공덕역 서울창업허브(https://naver.me/FjbnfZuu)
    - 매주 대관 상황에 따라 변동 가능
    - 대관 비용 나누어 지출(1만원 내)
    - 참석 인원 적을 경우 스터디룸 대여 가능
- `방식`: 온/오프라인 하이브리드 방식
    - 스터디 2주전 참석여부 투표에 따라 오프라인 인원 확정 및 대관 진행(발표자는 오프라인 참석 권장)
    - 오프라인 스터디 시 디스코드로 화면 공유하며 발표 및 이야기, 온라인 참여자는 디스코드 접속하여 참여
    - 발표 최소 1번 권장
    - 매주 Github에 스터디 과제 PR 올리고 Comment 남기기(최소 2개)
    - 발표자는 스터디 전주에 지원받음

### Rules

1. 사전 전달 없이 늦지 않기, 늦참/불참 시 언제든 연락 💌
2. 매주 과제 수행
3. 매주 발표자들의 PR에 Comment 달기(최소 2개)

## 스터디 커리큘럼

| 주차 | 주제 | 세부 내용 |
| --- | --- | --- |
| Week 1 | Intro & Icebreaking 🧊 | [스터디 세부 규칙 설정/아이스 브레이킹]<br><br>1. 스터디 규칙 설정<br>2. 스터디 일정 확정<br>3. 친해지기<br>4. 다음 주차 과제 소개<br>  • 개념 학습/실습 기록 |
| Week 2 | 장애를 보는 눈 👀  | [서비스 관측 기초 다지기]<br><br>1. 개념 학습<br>  • SLO/SLI/SLA와 관측 3종 세트 Metrics / Logs / Traces 알아보기<br><br>2. Loki + Grafana로 로그 수집/검색/대시보드 구축 경험 공유<br>  • 이론적 구성도 OK, 부담 NO<br>  • 현직자의 경우 실무 경험을 공유한다면 더 좋습니다 🙌<br>  • 다른 툴도 환영<br><br>3. 다음 주차 과제 소개<br>  • ‘알람 울렸을 때 10분 대응 플로우’ 런북 만들기(Notion 1페이지 내)<br>  • 스터디 시작 전 서로의 런북에 Comment 달기 |
| Week 3 | K8S 대표 장애 1️⃣ : 롤링 업데이트 실패 & 리소스 이슈 | [K8S 대표 장애 1]<br><br>1. 기초 학습<br>  • RollingUpdate 전략의 동작 방식<br>  • maxUnavailable / maxSurge<br>  • readinessProbe 실패가 트래픽에 미치는 영향<br>  • 잘못된 이미지 배포 시 어떤 일이 벌어지는가?<br>  • CPU Throttling & Memory Limit의 실제 영향<br><br>2. 장애 시나리오 공유/재현<br>  • 잘못된 이미지 배포, 새 버전에서 500 발생, readiness 통과 → 실제 트래픽 에러 발생<br>  • 관측 포인트: 5xx 증가, Pod restart 없음, Deployment rollout status<br><br>3. 다음 주차 과제 소개<br>  • 개념 학습<br>  • Node 장애 시나리오, MTTR 목표값 설정한 K8S 장애 시나리오 런북 작성<br>  • “노드 한 대가 죽었을 때” 어떤 알람이 먼저 울려야 하는가? |
| Week 4 | 좋은 알람의 조건 ⏰ | [관측에서 행동으로]<br><br>1. 좋은 장애 알람이란?<br>  • 각자가 생각하는 좋은 알람의 조건에 대해 자유롭게 공유해봅시다.<br>  • MTTR을 최소화하기 위한 고민을 공유해봅시다.<br><br>2. 알람 설계 경험 공유 / 런북 피드백<br>  • Week 2에서 만든 대시보드/SLI 기반으로 알람 설계<br>  • 런북 공유, 피드백<br> <br>3. 다음 주차 과제 소개<br>  • 개념 학습<br>  • DB Pool 고갈 런북 작성(본인 서비스/프로젝트의 DB 장애 경험 바탕, 없다면 시나리오 제작) |
| Week 5 | K8S 대표 장애 2️⃣ : Node 장애 + 스케줄링 실패 + 클러스터 레벨 이슈 | [K8S 대표 장애 1]<br><br>1. 기초 학습<br>  • Node NotReady의 의미<br>  • cordon / drain 차이<br>  • PDB가 MTTR에 미치는 영향<br>  • anti-affinity / topology spread<br>  • Replica 수가 부족할 때 벌어지는 일<br><br>2. 장애 시나리오 공유<br>  • 시나리오 A: Node 한 대 Down<br>  • 일부 Pod가 사라짐 / 재스케줄링 지연 /PDB 때문에 drain 지연<br>  • 시나리오 B: 스케줄링 실패<br>  • Pending 상태 지속 / 자원 부족 / affinity 조건 충돌<br>  • 시나리오 C: DNS 장애<br>  • 외부 API 호출 실패 / DB 연결 실패 / 실제 Pod는 정상<br><br>3. 다음 주차 과제 소개<br>  • 개념 학습<br>  • External API/네트워크 지연 시나리오 런북 작성 |
| Week 6 | 커넥션 풀 말리기 🫥 | [DB Connection Pool 고갈]<br><br>1. DB Connection Pool이란?<br>  • 커넥션 풀이 고갈되면 왜 연쇄 장애가 발생할까?<br>  • p95/p99 latency 상승 → DB timeout → 5xx 증가의 원인<br><br>2. 대응 경험/이론 학습 공유<br>  • 관측 포인트 & 대시보드/로그로 징후 찾기<br>  • 런북 공유, 피드백<br><br>3. 다음 주차 과제 소개<br>  • 개념 학습<br>  • Pod CrashLoop / OOMKill - K8S 장애 시나리오 런북 작성 |
| Week 7 | 스터디 진행상황 보고 결정<br>~~내 탓이 아닌 장애 🌐~~ | ~~[External API/네트워크 지연]~~ |
| Week 8 | 스터디 회고 | 회고 |
