# 🧱 기초 개념

## 1. SLO / SLI / SLA란?

이 세 가지는 모두 **서비스 품질을 어떻게 정의하고 관리할 것인가**와 관련된 개념입니다.
쉽게 말하면,

* **SLI**는 “무엇을 측정할 것인가”
* **SLO**는 “어느 수준까지를 목표로 할 것인가”
* **SLA**는 “그 목표를 못 지켰을 때 어떤 약속과 책임이 있는가”

## 1-1. SLI (Service Level Indicator)

**서비스의 상태를 수치로 측정하는 지표**입니다.
즉, “우리 서비스가 지금 얼마나 잘 동작하고 있는가?”를 측정하는 값입니다.

예를 들어 다음과 같은 것들이 SLI가 될 수 있습니다.

* 요청 성공률
* 응답 시간(latency)
* 에러율(error rate)
* 가용성(availability)
* 특정 작업의 처리 시간
* 메시지 큐 적체량

### 예시

* 최근 5분간 `/api/search` 요청 성공률: **99.3%**
* p95 응답 시간: **420ms**
* 한 시간 동안 500 에러 비율: **0.8%**

## 1-2. SLO (Service Level Objective)

**SLI에 대해 팀이 목표로 정한 기준치**입니다.
즉, “이 정도 수준은 유지하자”라고 정한 운영 목표입니다.

### 예시

* 월간 가용성 **99.9% 이상 유지**
* p95 응답 시간 **500ms 이하 유지**
* 결제 API 성공률 **99.95% 이상 유지**

여기서 중요한 점은,
SLO는 단순히 “좋으면 좋다”가 아니라 **운영 판단 기준**이 된다는 것입니다.

예를 들어:

* 에러 예산(error budget)을 얼마나 썼는지 판단
* 지금 기능 개발보다 안정화가 우선인지 판단
* 알람 기준을 어디에 둘지 결정
* 장애 대응 우선순위를 정하는 기준

## 1-3. SLA (Service Level Agreement)

**서비스 제공자와 사용자(또는 고객) 사이의 공식적인 약속**입니다.
SLO가 내부 목표라면, SLA는 외부 계약 또는 대외 약속에 가깝습니다.

SLA에는 보통 다음이 포함됩니다.

* 보장하는 서비스 수준
* 측정 방법
* 예외 조건
* 미달 시 보상 또는 책임

### 예시

* “월 가용성 99.9% 미만일 경우 요금의 일부를 환불합니다.”
* “영업시간 내 장애 문의는 1시간 내 응답합니다.”

## 1-4. 요약

| 개념  | 의미    | 질문                   |
| --- | ----- | -------------------- |
| SLI | 측정 지표 | 무엇을 측정할 것인가?         |
| SLO | 목표 기준 | 어느 정도 수준을 목표로 할 것인가? |
| SLA | 외부 약속 | 못 지키면 어떤 책임을 질 것인가?  |

# 2. Metrics / Logs / Traces란?

이 세 가지는 **Observability(관측 가능성)** 의 핵심 데이터입니다.
장애를 빨리 인지하고, 원인을 좁히고, 복구하기 위해 사용합니다.

쉽게 구분하면:

* **Metrics**: 숫자로 보는 전체 상태
* **Logs**: 사건별로 남는 기록
* **Traces**: 요청 하나가 시스템을 지나간 경로

## 2-1. Metrics

**시스템이나 서비스 상태를 숫자로 집계한 시계열 데이터**입니다.
시간에 따라 변하는 값을 그래프로 보는 데 적합합니다.

### 대표 예시

* CPU 사용률
* 메모리 사용량
* 요청 수(RPS)
* 에러율
* 응답 시간(p95, p99)
* DB connection 수
* 큐 길이

### Metrics가 잘하는 일

* 전체 상태를 빠르게 파악
* 이상 징후 감지
* 알람 조건 설정
* 추세 확인

### 예시 질문

* 지금 트래픽이 얼마나 늘었는가?
* 에러율이 언제부터 증가했는가?
* 응답 시간은 어느 시점에 튀었는가?

## 2-2. Logs

**시스템 내부에서 발생한 이벤트를 텍스트 또는 구조화된 형태로 기록한 데이터**입니다.

예를 들면:

* 에러 메시지
* 예외 stack trace
* 요청/응답 정보
* 사용자 행동 기록
* 배포 로그
* 쿼리 실행 로그

### 로그 예시

```text
2026-03-20T18:02:11 ERROR SearchService - DB connection acquire timeout
requestId=ab12cd34 userId=1024 endpoint=/api/search
```

### Logs가 잘하는 일

* 구체적으로 무슨 일이 발생했는지 확인
* 에러 메시지와 예외 추적
* 특정 사용자나 요청 단위 조사
* 디버깅

### 예시 질문

* 정확히 어떤 에러가 났는가?
* 어느 endpoint에서 에러가 발생했는가?
* 특정 사용자의 요청에서 무슨 일이 있었는가?

## 2-3. Traces

**하나의 요청이 여러 시스템을 거치며 처리되는 전체 흐름을 추적한 데이터**입니다.

특히 마이크로서비스나 외부 API, DB 호출이 여러 단계로 이어질 때 유용합니다.

예를 들어 사용자 요청 하나가 다음처럼 흘러갈 수 있습니다.

* API Gateway
* User Service
* Search Service
* PostgreSQL
* Redis
* 외부 API

Trace는 이 요청이 각 구간에서 **얼마나 걸렸는지**, **어디서 느려졌는지**, **어디서 실패했는지**를 보여줍니다.

### Traces가 잘하는 일

* 병목 구간 찾기
* 서비스 간 호출 흐름 파악
* 느린 구간 식별
* 장애 전파 경로 확인

### 예시 질문

* 전체 요청 3초 중 어디서 2.4초를 썼는가?
* 앱이 느린 건지 DB가 느린 건지 외부 API가 느린 건지?
* 어떤 서비스 호출에서 timeout이 발생했는가?

## 2-4. 요약

| 개념      | 형태     | 잘하는 것           | 대표 질문       |
| ------- | ------ | --------------- | ----------- |
| Metrics | 숫자 시계열 | 이상 징후 감지, 추세 확인 | 지금 얼마나 느린가? |
| Logs    | 이벤트 기록 | 에러 원인 확인, 디버깅   | 무슨 에러가 났는가? |
| Traces  | 요청 흐름  | 병목 분석, 호출 경로 추적 | 어디서 느려졌는가?  |

# 💭 각자의 생각 (장애와 장애 대응에 대해)

완벽한 프로그램은 만들기 어렵다고 생각합니다.
비즈니스 환경에서는 요구사항이 계속 바뀌고, 처음부터 모든 경우를 예측해 처리하는 데에는 분명한 한계가 있기 때문입니다.

그리고 실제로 방치되는 문제들 중 많은 수는 기술 자체의 한계보다는 시간, 우선순위, 사람의 판단 같은 현실적인 이유에서 비롯된다고 생각합니다.
이 점은 사람이 건강의 중요성을 알면서도 늘 건강하게 살지 못하는 모습과도 닮아 있습니다.

그런 의미에서 사람의 병과 시스템의 장애는 비슷합니다.
문제는 갑자기 생기는 것처럼 보이지만, 실제로는 이전부터 징후가 있었을 가능성이 큽니다.

다만 사람과 달리 시스템은 스스로 자신의 상태를 느끼지 못합니다.
사람에게 감각이 당연한 것처럼 주어져 있다면, 시스템에게는 그런 감각을 우리가 직접 만들어주어야 합니다.

그래서 장애 대응에서 관찰 가능성은 핵심이라고 생각합니다.
관찰 가능성을 잘 설계한다는 것은 장애 이후를 보는 것이 아니라, 장애 가능성을 이해하고 그 징후를 미리 관측할 수 있도록 준비하는 일에 가깝습니다.

최근에는 AI를 통해 장애 시나리오를 더 다양하게 가정하고, 더 빠르게 탐지하고 해석할 수 있게 되었습니다.
결국 장애 대응은 기술만의 문제가 아니라, 시스템에 얼마나 관심을 기울이고 꾸준히 들여다보느냐의 문제라고 생각합니다.

관심을 기울인 만큼 더 빠르게 징후를 발견할 수 있고, 더 유연하게 대응할 수 있으며, 더 견고한 시스템을 만들어갈 수 있다고 생각합니다.

# 📝 장애 대응 플로우 런북

> GPT의 멱살을 잡고 만든 가상 시나리오

**트래픽 증가 이후 API 응답 지연 및 500 에러 급증**

## 1️⃣ 장애 시나리오

주말 저녁, 프로모션성 기능이 외부 커뮤니티에 공유되면서 서비스 트래픽이 평소 대비 약 4배 증가했다.
이후 사용자들이 메인 페이지와 검색 기능 사용 시 응답이 매우 느려지거나, 일부 요청에서 500 에러를 경험하기 시작했다.

서비스는 다음과 같은 환경에서 운영되고 있다.

* **환경**

  * 단일 EC2 인스턴스
  * docker-compose 기반 배포
  * Nginx
  * Application Server
  * PostgreSQL
  * Prometheus + Grafana
  * Loki(or ELK) 기반 로그 수집

* **문제가 발생한 서비스**

  * 메인 API 서버
  * 검색/목록 조회 API
  * DB 조회가 많은 엔드포인트

* **사용자 영향**

  * 페이지 로딩 지연
  * 검색 결과 미표시
  * 간헐적 500 Internal Server Error 발생
  * 일부 사용자는 “서비스가 멈춘 것 같다”고 인식

## 2️⃣ 장애 인지

장애는 다음 경로를 통해 인지되었다.

* **Grafana Alert**

  * API p95 latency가 300ms → 4s 이상으로 급증
  * error rate가 1% 미만 → 18%까지 상승
  * DB connection usage가 90% 이상으로 지속
* **Slack 알람**

  * `High API Latency`
  * `HTTP 5xx Error Rate Increased`
  * `DB Connection Pool Saturation`
* **유저 문의**

  * “검색이 안 돼요”
  * “페이지가 너무 느려요”
  * “새로고침하면 에러가 떠요”

### 판단에 사용한 주요 지표

* **Application**

  * Request count / RPS
  * p50 / p95 / p99 latency
  * HTTP 5xx error rate
* **Infrastructure**

  * CPU usage
  * Memory usage
  * Network traffic
* **Database**

  * DB connection pool usage
  * Slow query count
  * Query execution time
* **로그**

  * timeout 관련 에러
  * DB connection acquire timeout
  * 특정 API endpoint 집중 에러

## 3️⃣ 대응 플로우

장애를 인지한 직후에는 “어디가 병목인지”를 가장 빠르게 좁히는 것이 중요하다.
이번 시나리오에서는 **사용자 영향도 확인 → 시스템 상태 확인 → 최근 변경점 확인 → 로그/DB 확인** 순서로 대응한다.

### 1. Grafana Dashboard 확인

가장 먼저 전체 시스템 상태를 확인한다.

* 확인 항목

  * traffic 증가 여부
  * latency 상승 여부
  * error rate 상승 여부
  * CPU / Memory / Network 상태
  * DB connection pool usage

### 2. 어떤 API가 문제인지 확인

전체 장애인지, 특정 기능 장애인지 범위를 좁힌다.

* endpoint별 latency 확인
* endpoint별 5xx 비율 확인
* 특정 API(`/search`, `/feed`, `/courses` 등) 집중 여부 확인

### 3. 최근 배포 여부 확인

장애 직전 변경 사항이 있었는지 확인한다.

* 최근 배포 시간 확인
* CI/CD 로그 확인
* 애플리케이션 설정 변경 여부 확인
* feature flag / 환경변수 변경 여부 확인

### 4. 애플리케이션 로그 검색

로그에서 공통 패턴을 찾는다.

예시 검색 키워드

```text
timeout
connection acquire timeout
too many clients
500
exception
database
```

확인 포인트

* DB connection timeout 발생 여부
* 특정 API 호출에서 반복되는 stack trace 존재 여부
* 외부 API timeout 여부
* N+1 조회 혹은 비정상 반복 쿼리 징후

### 5. 컨테이너 / 프로세스 상태 확인

애플리케이션 자체가 비정상 상태인지 확인한다.

```bash
docker ps
docker stats
docker compose logs app --tail=200
docker compose logs nginx --tail=200
docker compose logs db --tail=200
```

확인 포인트

* app 컨테이너 재시작 여부
* 메모리 급증 여부
* CPU 과다 사용 여부
* nginx upstream timeout 여부

### 6. DB 상태 확인

DB 병목이 의심되면 바로 확인한다.

* connection pool 포화 여부
* active query 수
* slow query 존재 여부
* lock 발생 여부
* 특정 테이블/쿼리 집중 여부

예시 확인 항목

```sql
-- 현재 실행 중인 쿼리 확인
SELECT pid, usename, state, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- 오래 걸리는 쿼리 확인
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### 7. 원인 후보 정리

이 시점에서 가능한 원인을 분류한다.

* 트래픽 증가로 인한 단순 리소스 부족
* 느린 쿼리로 인한 DB connection pool 고갈
* 최근 배포된 코드의 비효율 쿼리
* 외부 API 지연이 내부 timeout으로 전파
* 캐시 부재로 DB 직접 조회 과다

## 4️⃣ 초기 대응

이번 장애의 초기 대응 목표는 **원인 완전 해결**이 아니라,
우선 **사용자 영향도를 빠르게 줄이고 서비스 복구**하는 것이다.

### 즉시 가능한 완화 방법

#### A. 문제 기능 임시 제한

부하가 집중되는 검색/추천 기능을 임시 비활성화한다.

* feature flag off
* 무거운 정렬/필터 기능 제거
* 실시간 추천 API fallback 처리

#### B. 최근 배포 롤백

장애 직전 배포 이후 문제가 시작되었다면 빠르게 롤백한다.

* 직전 안정 버전으로 복구
* 신규 쿼리 로직 제거
* 실험 기능 비활성화

#### C. 애플리케이션 재기동

connection pool 또는 애플리케이션 내부 리소스가 꼬인 경우 임시 완화용으로 수행한다.

```bash
docker compose restart app
```

주의:

* 재기동은 근본 원인 해결이 아님
* 순간적으로 복구되어도 재발 가능

#### D. 트래픽 완화

* Nginx rate limiting 적용
* 특정 크롤러 / 비정상 요청 차단
* 내부 관리자 기능 일시 차단

#### E. DB 부하 완화

* 무거운 API를 임시 차단
* pagination size 축소
* 비필수 JOIN 제거된 fallback API 사용

## 5️⃣ 원인 분석

최종 원인은 다음과 같다고 가정할 수 있다.

### 장애 원인

프로모션 유입으로 검색 API 호출이 급증했고,
해당 API 내부에서 최근 배포된 쿼리 로직이 비효율적으로 동작하면서 **N+1 조회와 긴 실행 시간**을 유발했다.

이로 인해:

1. 요청당 DB 쿼리 수가 급증했고
2. 쿼리 실행 시간이 길어졌으며
3. DB connection pool이 빠르게 포화되었고
4. 애플리케이션이 DB connection을 제때 획득하지 못해 timeout 발생
5. 결국 API latency 상승과 500 에러 증가로 이어졌다

### 근거

* 특정 endpoint에서만 latency 급증
* 장애 직전 검색 로직 관련 배포 이력 존재
* 로그에 `connection acquire timeout` 반복 발생
* DB slow query 로그에서 해당 API 관련 query 확인
* pg_stat_statements 상 특정 query mean_exec_time 급증 확인

## 6️⃣ 사후 방지 대책 🚀

이번 장애가 다시 발생하지 않도록 하기 위해 다음과 같은 개선을 진행한다.

### 1. SLO 기반 알람 체계 강화

단순 CPU 알람이 아니라 사용자 경험 중심 알람으로 개선한다.

* API p95 latency 기준 alert
* 5xx error rate 기준 alert
* DB connection pool saturation alert
* endpoint별 latency / error dashboard 분리

### 2. 쿼리 최적화 및 성능 점검 체계 추가

* N+1 방지 코드 리뷰 체크리스트 추가
* 쿼리 실행 계획(EXPLAIN ANALYZE) 검토
* 인덱스 재점검
* 대량 조회 API pagination 강제

### 3. 배포 전 성능 검증 절차 도입

* staging 환경 부하 테스트
* k6 / JMeter 기반 간단한 smoke load test
* 주요 API latency regression 체크

### 4. 기능 단위 feature flag 적용

문제 기능만 빠르게 끌 수 있도록 설계한다.

* 검색 기능 flag
* 추천 기능 flag
* 무거운 통계/집계 API flag

### 5. Dashboard 개선

“전체 상태”만 보는 대시보드가 아니라, 장애 판단에 필요한 계층별 시야를 만든다.

* Service overview dashboard
* Endpoint별 latency dashboard
* DB health dashboard
* Deployment / release annotation 추가

### 6. 로그 / 트레이싱 강화

* request id 기반 추적
* endpoint별 DB query count 로깅
* slow query threshold 초과 시 구조화 로그 남기기
* tracing으로 app → DB 구간 병목 확인

### 7. 아키텍처 개선 검토

스터디 심화 주제로도 연결 가능하다.

* 단일 EC2 → scale-out 가능한 구조 검토
* 읽기 부하가 큰 API 캐시 도입 검토
* DB read replica 검토
* connection pool 설정 재조정
