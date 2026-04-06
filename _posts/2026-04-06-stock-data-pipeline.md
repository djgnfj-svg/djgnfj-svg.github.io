---
title: "Airflow 기반 주식 데이터 파이프라인 설계·운영기 — 수집부터 LLM 판단까지"
date: 2026-04-06 00:00:00 +0900
categories: [Data Engineering]
tags: [airflow, data-pipeline, python, langgraph, llm, postgresql, monitoring, grafana, prometheus, docker]
---

## TL;DR

진입장벽이 낮은 영역에 AI를 붙여 빠르게 수익을 만드는 것이 목표였고, 그 첫 번째 대상으로 한국 주식을 선택했다. 매일 장 마감 후 시세·재무·경제지표·뉴스를 6개 소스에서 자동 수집하고, PostgreSQL 3계층 스키마(raw/staging/mart)로 정제·적재한 뒤, LangGraph 4-에이전트(Quant→Qual→RedTeam→Judge) 토론 구조로 AI가 투자 판단을 내리는 파이프라인을 Airflow 4개 DAG로 설계·운영한 기록이다. Prometheus + Grafana로 메트릭을 수집하고, Slack으로 장애 알림을 보낸다. 전체 인프라는 Docker Compose 6컨테이너로 구성했다. 기술 선택마다 "왜"를 중심으로 서술했다.

---

## 1. 프로젝트 배경 & 전체 아키텍처

### 동기

진입장벽이 낮은 영역에 AI를 붙여 빠르게 수익을 만들고 싶었다. 그 첫 번째 대상으로 한국 주식을 선택했다. 크롤링 경험과 Python 백엔드 개발 경험을 바로 살릴 수 있는 도메인이었고, 주식 데이터는 정형화되어 있어 파이프라인 자동화에 적합했기 때문이다.

수집 자체는 어렵지 않았다. 문제는 그 이후였다.

야근하거나 잠시 놓친 날에 데이터 공백이 생겼다. 잠들기 전에 컴퓨터를 켜두자니 소음이 거슬렸다.

클라우드에 올리면 해결되지만, 아직 수익이 증명되지 않은 단계에서 인프라 비용을 태우고 싶지 않았다. 로컬에서 먼저 파이프라인을 구축하고 검증한 뒤, 성과가 확인되면 클라우드로 올리기로 했다. 현재는 AWS로 이전을 준비 중이다.

수집부터 AI 판단까지 전체 흐름을 파이프라인으로 묶는 것이 첫 번째 과제였다.

### 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Docker Compose (6 컨테이너)                       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Airflow (Scheduler + Webserver)                 │    │
│  │                                                             │    │
│  │  DAG 1: daily_collect (19:30 KST)                          │    │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐   │    │
│  │  │시세    │ │재무    │ │경제지표│ │뉴스    │ │글로벌  │   │    │
│  │  │pykrx  │ │DART   │ │ECOS   │ │Naver  │ │yfinance│   │    │
│  │  │yfinance│ │       │ │       │ │API    │ │Tavily │   │    │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘   │    │
│  │      └──────────┴─────┬────┴──────────┴──────────┘        │    │
│  │                       ▼                                    │    │
│  │              ┌──────────────┐                              │    │
│  │              │  정제 & 검증  │                              │    │
│  │              └──────┬───────┘                              │    │
│  │                     ▼                                      │    │
│  │  ┌──────────────────────────────────┐                     │    │
│  │  │  PostgreSQL 16 (3계층 스키마)     │                     │    │
│  │  │  raw → staging → mart            │                     │    │
│  │  └──────────────┬───────────────────┘                     │    │
│  │                 ▼                                          │    │
│  │  DAG 2: daily_predict (05:00 KST)                         │    │
│  │  ┌──────────────────────────────────┐                     │    │
│  │  │  LangGraph 4-Agent Debate        │                     │    │
│  │  │  Quant → Qual → RedTeam → Judge  │                     │    │
│  │  └──────────────┬───────────────────┘                     │    │
│  │                 ▼                                          │    │
│  │  DAG 3: daily_verify (16:00 KST)                          │    │
│  │  DAG 4: weekly_backtest (토 10:00 KST)                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ PostgreSQL 16│  │ Prometheus   │  │ Grafana      │              │
│  │ :5432        │  │ :9090        │  │ :3000        │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                     │
│  Slack ← 장애 알림                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

| 컴포넌트 | 역할 | 선택 이유 |
|---------|------|----------|
| Airflow | 오케스트레이션 | Task 간 의존성 관리, 실패 재시도, 4개 DAG 스케줄링을 하나의 도구로 |
| pykrx + yfinance | 시세 수집 | pykrx는 KOSPI 매매동향/공매도, yfinance는 글로벌 지수/환율/원자재 |
| DART API | 재무 수집 | 금감원 공식 공시 데이터, 신뢰도 높음 |
| ECOS API | 경제지표 수집 | 한국은행 공식 API, 국내 경제지표의 가장 정확한 소스 |
| Naver API + Tavily | 뉴스 수집 | 국내 뉴스는 Naver, 글로벌 인텔리전스는 Tavily로 보강 |
| PostgreSQL 16 | 저장소 | Upsert 네이티브 지원, JSONB, 3계층 스키마 운영 |
| LangGraph | AI 판단 | 단일 LLM 호출이 아닌 4-에이전트 토론 구조 |
| Prometheus + Grafana | 모니터링 | 메트릭 수집 + 대시보드 시각화 |
| Slack | 알림 | 즉시 확인 가능, 장애 대응 속도 |

이 구조의 핵심은 **각 단계가 독립적으로 실패하고 복구할 수 있는 것**이다.

수집이 실패해도 어제 데이터로 AI 판단은 돌릴 수 있다. AI 판단이 실패해도 수집·적재는 정상 완료로 기록된다.

DAG를 4개로 분리한 이유도 같다. 수집과 예측은 실행 시점과 실패 특성이 다르다. 하나의 DAG에 몰아넣으면 운영이 복잡해진다.

---

## 2. 데이터 수집 설계

### 수집 원칙

| 원칙 | 적용 방식 | 이유 |
|------|----------|------|
| 수집과 파싱 분리 | raw 스키마에 JSONB로 원본 보존, staging에서 정제 | 변환 로직이 바뀌어도 재처리 가능 |
| Graceful Degradation | 개별 종목/API 실패해도 수집 계속 | 500개 중 3개 실패로 전체를 멈출 수 없다 |
| 멱등성 보장 | UPSERT 패턴, (date, ticker) 유니크 키 | 같은 수집을 두 번 돌려도 데이터가 꼬이지 않는다 |
| Rate Limiting | 배치 딜레이 + 지수 백오프 | API 차단 방지 |
| 의존성 버전 고정 | 핵심 라이브러리 엄격 고정 | 라이브러리 업데이트로 응답 구조가 바뀌는 사고 방지 |

### 6개 소스, 역할 분담

| 소스 | 수집 데이터 | 특징 |
|------|------------|------|
| pykrx | KOSPI 시총 상위 종목, 매매동향, 공매도 | 국내 특화 데이터 |
| yfinance | 주가 OHLCV, 글로벌 지수, 원자재, 환율 | 글로벌 맥락 보강 |
| DART API | 기업 공시, 재무제표 | 금감원 공식, 신뢰도 높음 |
| ECOS API | 금리, CPI, GDP 등 국내 경제지표 | 한국은행 공식 |
| Naver API | 국내 뉴스 | CLIENT_ID/SECRET 기반 |
| Tavily API | 글로벌 뉴스, 인텔리전스 | 국내 뉴스의 사각지대 보강 |

소스를 6개로 분산한 이유는 단일 소스 의존 리스크를 줄이기 위해서다.

pykrx가 장애를 일으켜도 yfinance로 기본 시세는 확보할 수 있다. Naver API가 막혀도 Tavily로 뉴스를 가져올 수 있다.

### 수집 코드 패턴

```python
with ThreadPoolExecutor(max_workers=5) as executor:
    for i in range(0, len(symbols), batch_size):
        batch = symbols[i:i + batch_size]
        futures = {executor.submit(collect_single, s): s for s in batch}
        for future in as_completed(futures):
            symbol = futures[future]
            try:
                result = future.result()
                success_count += 1 if result else failed.append(symbol)
            except Exception as e:
                logger.error(f"{symbol} 수집 실패: {e}")
                failed.append(symbol)
        time.sleep(delay)
```

`ThreadPoolExecutor`를 쓴 이유는 pykrx/yfinance 내부가 동기 HTTP 호출이라 `asyncio`의 이점이 제한적이기 때문이다. I/O 대기 시간을 스레드로 겹치는 것만으로 충분했다.

배치 간 딜레이는 API rate limit 대응이다.

---

## 3. Airflow 파이프라인 설계

이 프로젝트의 핵심이다. "왜 Airflow인가"부터 실제 운영에서 겪은 문제까지.

### 왜 Airflow인가

| 기준 | Cron | Airflow | Prefect | Dagster |
|------|------|---------|---------|---------|
| Task 의존성 관리 | 불가 (스크립트로 직접 구현) | 네이티브 | 네이티브 | 네이티브 |
| 실패 재시도 | 직접 구현 | 내장 (retries, backoff) | 내장 | 내장 |
| 실행 이력 / UI | 없음 | 웹 UI 제공 | 클라우드 UI | 웹 UI |
| 백필(과거 데이터 재처리) | 수동 | catchup + 날짜 파라미터 | 가능 | 가능 |
| 생태계 / 레퍼런스 | - | 압도적 | 성장 중 | 성장 중 |
| 러닝 커브 | 낮음 | 중간 | 낮음 | 중간 |
| 채용 시장 수요 | - | 데이터 엔지니어 공고 대부분 언급 | 일부 | 일부 |

처음에는 Cron + 쉘 스크립트로 시작했다.

수집 → 검증 → 적재가 순차적으로 실행되어야 하는데, Cron에서는 "앞 단계가 성공했을 때만 다음 단계 실행"을 구현하려면 lock 파일이나 상태 파일을 직접 관리해야 했다. 실패 시 재시도도 직접 구현해야 했고, 어떤 Task가 언제 실패했는지 추적하기 어려웠다.

Airflow를 선택한 결정적 이유는 세 가지다:

1. **Task 간 의존성을 코드로 선언**할 수 있다. `>>` 연산자 한 줄이면 된다.
2. **실패 재시도와 알림이 내장**되어 있다. 설정만 하면 된다.
3. **실행 이력과 로그를 웹 UI에서 확인**할 수 있다. 새벽에 실패한 파이프라인을 아침에 UI에서 바로 확인할 수 있다.

Prefect나 Dagster도 검토했다.

Prefect는 러닝 커브가 낮지만 Prefect Cloud 의존도가 높았다. 로컬 운영에서 레퍼런스가 부족했다. Dagster는 Asset 기반 패러다임이 흥미로웠으나, 이 프로젝트의 Task 기반 흐름에는 Airflow가 더 직관적이었다.

### 4개 DAG 설계

DAG를 4개로 분리한 것이 이 파이프라인의 핵심 설계 결정이다.

| DAG | 스케줄 (KST) | 역할 | 분리 이유 |
|-----|-------------|------|----------|
| `daily_collect` | 평일 19:30 | 수집 → 정제 → 적재 | 장 마감(15:30) 후 데이터 확정 대기 |
| `daily_predict` | 평일 05:00 | LangGraph AI 판단 | 수집과 분리: LLM 실패가 수집에 영향 주지 않도록 |
| `daily_verify` | 평일 16:00 | 전일 예측 vs 실제 결과 검증 | 장 마감 후 실제 데이터로 사후 검증 |
| `weekly_backtest` | 토요일 10:00 | 주간 백테스트 | 과거 데이터 기반 전략 검증, 주 1회면 충분 |

**왜 하나의 DAG에 몰아넣지 않았는가**

수집(19:30)과 예측(05:00)의 실행 시점이 다르다. 하나로 묶으면 수집 후 9시간을 기다려야 한다. LLM 장애로 예측이 실패했을 때 수집까지 재실행되는 것도 방지할 수 있다.

**daily_verify가 존재하는 이유**

AI 판단의 정확도를 추적하지 않으면 모델이 잘하고 있는지 알 수 없다. 매일 전일 예측과 실제 결과를 비교하여 정확도를 기록한다. 이 데이터가 쌓이면 weekly_backtest에서 전략 개선의 근거가 된다.

### daily_collect DAG 상세

```python
default_args = {
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(minutes=30),
    'execution_timeout': timedelta(minutes=60),
}

with DAG(
    'daily_collect',
    default_args=default_args,
    schedule='30 10 * * 1-5',  # UTC 10:30 = KST 19:30
    catchup=False,
    max_active_runs=1,
    tags=['stock', 'collect', 'production'],
) as dag:

    # 수집 — 5개 Task 병렬
    collect_prices = PythonOperator(task_id='collect_prices', ...)      # pykrx + yfinance
    collect_fundamentals = PythonOperator(task_id='collect_fundamentals', ...)  # DART
    collect_economic = PythonOperator(task_id='collect_economic', ...)   # ECOS
    collect_news = PythonOperator(task_id='collect_news', ...)           # Naver API
    collect_global = PythonOperator(task_id='collect_global', ...)       # yfinance + Tavily

    # 정제 & 검증
    transform = PythonOperator(
        task_id='transform_and_validate',
        trigger_rule=TriggerRule.ALL_SUCCESS,
    )

    # 적재
    load = PythonOperator(task_id='load_to_postgresql', ...)

    # 알림 — 성공이든 실패든 항상 실행
    notify = PythonOperator(
        task_id='send_notification',
        trigger_rule=TriggerRule.ALL_DONE,
    )

    [collect_prices, collect_fundamentals, collect_economic, collect_news, collect_global] >> transform >> load >> notify
```

**Task 분리 기준**: 각 Task가 독립적으로 실패하고 재시도될 수 있어야 한다. 시세 수집은 성공했는데 DART API만 장애일 때, 전체를 재실행할 필요가 없다.

**`max_active_runs=1`**: 이전 실행이 끝나기 전에 다음 실행이 시작되면 수집 데이터가 꼬인다.

**`catchup=False`**: Airflow를 재시작했을 때 밀린 스케줄을 한꺼번에 실행하지 않는다. 과거 데이터 소급은 별도 백필로 처리한다.

### 실행 전략: 재시도와 실패 처리

```python
default_args = {
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,   # 5분 → 10분 → 20분
    'max_retry_delay': timedelta(minutes=30),
}
```

재시도를 2회로 설정한 이유: API 일시 장애는 대부분 10분 내에 복구된다. 3회 이상이면 근본적인 문제일 가능성이 높아 알림을 보내는 게 낫다.

**`trigger_rule` 활용**

알림 Task에 `TriggerRule.ALL_DONE`을 설정했다. 기본값 `ALL_SUCCESS`는 앞 Task가 모두 성공해야 실행된다. `ALL_DONE`은 성공이든 실패든 실행된다.

파이프라인이 실패했을 때 알림이 안 가면 의미가 없다.

```python
def send_report(**context):
    dag_run = context['dag_run']
    failed_tasks = [
        ti.task_id for ti in dag_run.get_task_instances()
        if ti.state == 'failed'
    ]
    if failed_tasks:
        send_slack_alert(f"[FAIL] 실패 Task: {', '.join(failed_tasks)}")
    else:
        send_slack_message(f"[OK] 파이프라인 정상 완료")
```

### 운영 노하우

**DAG 버전 관리와 배포**: DAG 파일은 Git으로 관리하고, `dags/` 디렉토리에 심볼릭 링크를 걸었다. DAG 수정 후 Airflow가 파일 변경을 감지하면 자동으로 반영된다. DAG 파싱 에러로 전체 스케줄러가 멈추는 사고를 방지하기 위해 `dagbag_import_timeout`을 조정해두었다.

**성능 튜닝**:

| 설정 | 기본값 | 변경값 | 이유 |
|------|--------|--------|------|
| `parallelism` | 32 | 8 | 로컬 환경의 리소스 제약 (CPU 4코어) |
| `dag_concurrency` | 16 | 4 | 동일 DAG 내 동시 Task 수 제한 |
| `max_active_runs_per_dag` | 16 | 1 | 같은 DAG가 중복 실행되는 것 방지 |

로컬 환경이라 리소스가 제한적이다. 기본값 그대로 두면 수집 Task 5개가 동시에 ThreadPoolExecutor를 돌리면서 메모리가 부족해졌다. `parallelism`과 `dag_concurrency`를 낮춰서 해결했다.

**실제 겪은 운영 문제들**

**1) Scheduler 메모리 누수**

Airflow 2.x에서 Scheduler가 DAG 파싱을 반복하며 메모리가 점진적으로 증가했다. `AIRFLOW__SCHEDULER__NUM_RUNS`를 `1000`으로 설정해서 주기적으로 재시작하도록 했다.

**2) Task stuck**

네트워크 장애로 Task가 응답 없이 멈추는 경우가 있었다. `execution_timeout`을 설정하지 않으면 영원히 running 상태로 남는다. 모든 Task에 타임아웃을 설정하는 것이 필수다.

**3) DB 백엔드 교체**

처음에 Airflow 메타데이터 DB로 SQLite를 사용했다. `SequentialExecutor`만 사용 가능해서 Task 병렬 실행이 불가능했다. PostgreSQL로 교체하고 `LocalExecutor`를 사용해 해결했다.

---

## 4. 데이터 정제 & 적재

### 3계층 스키마 설계 (raw → staging → mart)

```
raw       — API 응답 원본 그대로 (JSONB)
staging   — 정제·정규화된 분석용 데이터
mart      — AI 판단 입력용 집계 데이터
```

**왜 3계층인가**

raw를 보존하는 이유는 단순하다. 이전 프로젝트에서 파서를 수정한 뒤 과거 데이터를 재처리할 수 없어 데이터 공백이 생긴 적이 있다. raw에 원본을 JSONB로 통째 저장해두면 변환 로직을 바꿔도 언제든 재처리할 수 있다.

staging과 mart를 분리한 이유는 관심사의 분리다. staging은 "정확하고 깨끗한 데이터", mart는 "AI 에이전트가 바로 소비할 수 있는 형태"다.

뉴스 데이터도 같은 패턴이다. raw에 Naver/Tavily API 응답을 JSONB로 통째 저장하고, staging에서 감성 분석 점수와 뉴스 분류를 추출한다. 두 API의 응답 구조가 다르기 때문에 raw 단계에서 공통 스키마를 강제하지 않는다.

### 스키마 예시

```sql
CREATE TABLE staging.stock_prices (
    symbol VARCHAR(10) NOT NULL,
    date DATE NOT NULL,
    open NUMERIC(12,4),
    high NUMERIC(12,4),
    low NUMERIC(12,4),
    close NUMERIC(12,4),
    volume BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (symbol, date)
);

CREATE INDEX idx_stock_prices_date ON staging.stock_prices(date);
CREATE INDEX idx_stock_prices_symbol ON staging.stock_prices(symbol);
```

`REAL` 대신 `NUMERIC(12,4)`를 사용한 이유는 부동소수점 오차 때문이다. 주가 비교 시 `0.1 + 0.2 != 0.3` 같은 문제가 분석 결과를 왜곡할 수 있다.

**인덱싱 전략**

`(symbol, date)` 복합 PK 외에 `date`와 `symbol` 단독 인덱스를 추가했다. "특정 날짜의 전 종목"과 "특정 종목의 히스토리" 조회가 모두 빈번하기 때문이다.

파티셔닝은 현재 규모(일일 수천 건, 누적 수십만 건)에서는 불필요하다. 수백만 건을 넘기면 date 기준 레인지 파티셔닝을 검토할 계획이다.

**Upsert 전략**:

```sql
INSERT INTO staging.stock_prices (symbol, date, open, high, low, close, volume)
VALUES (%s, %s, %s, %s, %s, %s, %s)
ON CONFLICT (symbol, date) DO UPDATE SET
    close = EXCLUDED.close,
    volume = EXCLUDED.volume,
    updated_at = NOW();
```

`ON CONFLICT DO UPDATE`를 선택한 이유는, 수정 종가나 거래량이 사후에 보정될 수 있기 때문이다. 새 데이터가 들어오면 갱신하는 것이 맞다.

### 데이터 품질 검증

적재 후 세 가지 검증을 자동으로 실행한다.

```python
def post_load_validation(execution_date):
    # 1. 건수 체크
    today_count = db.execute(
        "SELECT COUNT(*) FROM staging.stock_prices WHERE date = %s",
        [execution_date]
    ).scalar()
    if today_count < EXPECTED_MIN_COUNT * 0.9:
        alert(f"적재 건수 부족: {today_count}")

    # 2. null 비율
    null_ratio = db.execute("""
        SELECT COUNT(*) FILTER (WHERE close IS NULL)::float / COUNT(*)
        FROM staging.stock_prices WHERE date = %s
    """, [execution_date]).scalar()
    if null_ratio > 0.05:
        alert(f"close 필드 null 비율 이상: {null_ratio:.1%}")

    # 3. 전일 대비 급변 감지
    anomaly_count = db.execute("""
        SELECT COUNT(*) FROM staging.stock_prices t
        JOIN staging.stock_prices y ON t.symbol = y.symbol AND y.date = %s
        WHERE t.date = %s AND ABS(t.close - y.close) / y.close > 0.3
    """, [prev_date, execution_date]).scalar()
    if anomaly_count > 10:
        alert(f"전일 대비 30% 이상 변동 종목 {anomaly_count}건")
```

한국 주식은 가격제한폭이 ±30%이므로, 30% 초과 변동은 파싱 오류일 가능성이 높다. 미국 주식(제한폭 없음)과 다른 검증 기준이다.

---

## 5. LangGraph AI 판단

### 왜 LLM인가

처음에는 규칙 기반으로 시작했다. "RSI 30 이하면 매수 신호", "PER 업종 평균 이하면 저평가" 같은 규칙이다.

문제는 지표들이 서로 충돌할 때다. RSI는 매수 신호인데 거시 경제 지표는 침체를 가리키는 상황. 규칙 기반은 if-else가 폭발적으로 늘어난다. **다양한 데이터를 종합적으로 해석하는 능력**이 필요했고, 이건 LLM이 잘하는 영역이다.

단순 LLM API 호출이 아닌 LangGraph를 선택한 이유는 **멀티스텝 토론** 구조가 필요했기 때문이다. 한 명의 분석가에게 모든 걸 맡기면 확증 편향에 빠진다. 서로 다른 관점의 에이전트가 순차적으로 분석하고, 마지막에 종합 판단을 내리는 구조가 더 견고하다.

### 4-에이전트 토론 구조

```
┌──────────────────┐
│ Quantitative     │ ← 시세 + 기술적 지표 (RSI, MACD, 볼린저밴드)
│ Analyst          │
└──────┬───────────┘
       ▼
┌──────────────────┐
│ Qualitative      │ ← 뉴스 감성 + 거시 경제 + 재무 데이터
│ Analyst          │
└──────┬───────────┘
       ▼
┌──────────────────┐
│ Red Team         │ ← 위 두 분석의 약점을 공격, 반론 제기
└──────┬───────────┘
       ▼
┌──────────────────┐
│ Judge            │ ← 토론 결과 종합 → 최종 판단
└──────────────────┘
```

```python
graph = StateGraph(DebateState)
graph.add_node("quant", quantitative_analyst)   # 시세 + 기술적 지표
graph.add_node("qual", qualitative_analyst)     # 뉴스 + 거시 + 재무
graph.add_node("red_team", red_team)            # 반론 제기
graph.add_node("judge", judge)                  # 최종 판단

graph.set_entry_point("quant")
graph.add_edge("quant", "qual")
graph.add_edge("qual", "red_team")
graph.add_edge("red_team", "judge")
graph.add_edge("judge", END)

app = graph.compile()
```

각 에이전트는 LLM을 호출하여 자신의 역할에 맞는 분석을 수행하고, 결과를 State에 저장한다. 다음 에이전트는 이전 분석 결과를 참조하여 자신의 분석을 진행한다.

**왜 4개 에이전트인가**

핵심은 **Red Team**의 존재다. Quant와 Qual이 같은 방향의 결론을 내면 확증 편향이 생긴다. Red Team이 의도적으로 반론을 제기해서 Judge가 양쪽 논거를 모두 고려하게 한다.

실제 투자 기관에서 사용하는 "Devil's Advocate" 프로세스를 에이전트 그래프로 구현한 것이다.

각 노드를 분리한 또 다른 이유는 **프롬프트 전문화**다. 하나의 프롬프트에 모든 역할을 넣으면 분석 깊이가 얕아진다. 역할을 분리하면 각 노드의 프롬프트를 독립적으로 개선할 수 있고, 어느 단계에서 부정확한지 디버깅하기도 쉽다.

### 프롬프트 설계 전략

| 기법 | 내용 | 효과 |
|------|------|------|
| 출력 포맷 강제 | `{"signal": "매수", "confidence": 0.7, "reason": "..."}` | 파싱 안정성, 결과 DB 저장 용이 |
| 데이터 사전 요약 | raw 데이터 대신 핵심 통계만 전달 | 입력 토큰 60~70% 절감 |
| 판단 기준 명시 | "PER 15 이하를 저평가로 간주한다" | LLM이 기준을 만들어내는 것 방지 |
| 한국 시장 맥락 주입 | "가격제한폭 ±30%", "외국인 수급 비중 높음" | 미국 시장 기준 분석 방지 |

### Airflow와 LangGraph 연동

```python
def run_langgraph_agent(**context):
    market_data = fetch_latest_market_data()
    news_sentiment = fetch_latest_sentiment()
    economic_data = fetch_latest_economic_data()
    fundamental_data = fetch_latest_fundamentals()

    initial_state = {
        'market_data': market_data,
        'news_sentiment': news_sentiment,
        'economic_data': economic_data,
        'fundamental_data': fundamental_data,
    }

    try:
        result = app.invoke(initial_state)
        save_judgment(result['final_judgment'], context['execution_date'])
    except Exception as e:
        logger.error(f"LangGraph 실행 실패: {e}")
        # 아침 매수 → 저녁 매도 전략이므로, 판단 실패 시 투자를 스킵한다
        skip_today_trading(context['execution_date'])
        send_slack_alert(f"AI 이슈로 오늘 투자는 진행하지 못했습니다: {e}")
        raise
```

**타임아웃 관리**: LLM 호출은 응답 시간이 크게 변동한다. Airflow Task의 `execution_timeout=timedelta(minutes=30)`으로 상한을 둔다. 4개 노드 순차 실행 시 노드당 평균 2~3분, 전체 10~15분이 정상 범위다.

**Fallback 처리**: 이 시스템은 아침에 매수하고 저녁에 매도하는 데이트레이딩 전략이다. AI 판단이 실패하면 "전일 판단 유지"가 아니라 **오늘 투자 자체를 스킵**한다. 불확실한 판단으로 매수하는 것보다 하루를 쉬는 것이 리스크 관리상 더 안전하다. Slack으로 "AI 이슈로 오늘 투자는 진행하지 못했습니다"를 알리고, 수집·적재 데이터는 정상 보존되어 다음 날 판단에 활용된다.

**비용 관리**: LLM 호출은 매일 실행되므로 비용이 누적된다.

| 전략 | 효과 |
|------|------|
| 입력 데이터 사전 요약 (raw → 핵심 통계) | 입력 토큰 60~70% 절감 |
| 출력 포맷 제한 (장문 금지) | 출력 토큰 50% 절감 |
| 모델 혼합 (분석은 GPT-4o, 요약은 GPT-4o-mini) | 비용 30% 절감 |

### 결과 저장

```sql
CREATE TABLE mart.ai_judgments (
    id SERIAL PRIMARY KEY,
    execution_date DATE NOT NULL,
    signal VARCHAR(20),
    confidence NUMERIC(3,2),
    quant_analysis TEXT,
    qual_analysis TEXT,
    red_team_critique TEXT,
    final_reasoning TEXT,
    model_name VARCHAR(50),
    token_usage INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);
```

`model_name`과 `token_usage`를 저장하는 이유는 비용 추적과 모델 변경 시 판단 품질 비교를 위해서다. `daily_verify` DAG가 이 테이블의 과거 판단과 실제 주가 변동을 비교하여 정확도를 추적한다.

---

## 6. 모니터링 & 알람

### 모니터링 스택

```
Airflow (Task 실행 상태)
    │
    ├──→ Prometheus (메트릭 수집, :9090)
    │        │
    │        └──→ Grafana (대시보드, :3000)
    │               ├── stock_overview.json
    │               └── stock_detail.json
    │
    └──→ Slack (장애 알림)
```

Prometheus + Grafana를 별도로 구축한 이유는 Airflow UI만으로는 부족했기 때문이다. Airflow UI는 "이 Task가 성공/실패했는지"는 보여주지만, "수집 건수가 점진적으로 줄고 있는 추세"나 "LLM 응답 시간이 느려지는 경향"은 시계열 그래프로 봐야 파악된다.

### 모니터링 대상

| 카테고리 | 메트릭 | 알림 조건 |
|---------|--------|----------|
| 파이프라인 | DAG 실행 성공/실패 | 실패 시 즉시 |
| 파이프라인 | 전체 실행 시간 | 정상 대비 200% 초과 |
| 수집 | 수집 성공률 (성공/전체) | < 95% |
| 수집 | 수집 건수 | 전일 대비 30% 이상 감소 |
| 데이터 품질 | 필수 필드 null 비율 | > 5% |
| 데이터 품질 | 전일 대비 이상 변동 건수 | > 10건 |
| LLM | 응답 시간 | > 10분 (노드당) |
| LLM | 일일 토큰 사용량 | 예산 초과 시 |
| 인프라 | 컨테이너 상태 | 비정상 종료 시 |
| 인프라 | 디스크 사용률 | > 80% |

### 알람 체계

Airflow의 `on_failure_callback`으로 Task 실패 시 Slack에 자동 알림을 보낸다. 알림에는 실패한 Task 이름, 에러 메시지, Airflow 로그 링크를 포함한다.

알림 원칙:
- 실패 알림은 즉시, 성공 알림은 일일 리포트로 묶는다
- 동일 에러 반복 알림은 1시간 내 1회로 제한한다
- 알림에 로그 링크를 포함한다

### 실제 장애 사례

**사례 1: pykrx 데이터 지연**

장 마감 직후 pykrx 데이터가 아직 반영되지 않아 빈 데이터가 수집됐다. 건수 체크에서 감지. 수집 스케줄을 장 마감 4시간 후(19:30)로 조정하여 해결했다.

**사례 2: LLM API 응답 지연**

GPT-4 API 과부하로 AI 판단 Task가 30분 타임아웃에 걸렸다. 투자 스킵 처리되었고, Slack 알림으로 즉시 인지할 수 있었다.

**사례 3: PostgreSQL 디스크 풀**

데이터가 쌓이면서 디스크가 가득 찼다. 30일 이상 지난 raw 데이터를 아카이빙하는 배치를 추가하고, Prometheus에서 디스크 사용률 80% 이상 시 사전 알림을 보내도록 설정했다.

---

## 7. 회고 & 교훈

### 가장 어려웠던 점

Airflow 자체의 학습보다 **안정적으로 운영하는 것**이 어려웠다.

DAG 작성은 문서만 보면 된다. Scheduler 메모리 누수, Task stuck, 메타데이터 DB 성능 저하 같은 문제는 운영해봐야 겪는다. "쓸 줄 안다"와 "운영할 줄 안다"는 다른 차원의 역량이다.

### Airflow에 대한 솔직한 평가

**좋았던 점**: Task 의존성 관리, 실패 재시도, 웹 UI 모니터링은 기대 이상이었다. 새벽에 실패한 파이프라인을 아침에 UI에서 바로 확인할 수 있다.

**아쉬웠던 점**: 리소스 사용량이 생각보다 크다. 6개 컨테이너를 띄우면 유휴 상태에서도 메모리 2~3GB를 사용한다. 개인 프로젝트 규모에서는 무거운 것이 사실이다.

### LangGraph 4-에이전트 토론의 성과와 한계

**성과**: Red Team 에이전트의 도입이 가장 효과적이었다.

Quant와 Qual이 모두 "매수"를 외칠 때, Red Team이 "외국인 순매도 추세가 3일째 이어지고 있다"며 반론을 제기한다. Judge가 이를 반영해 "매수"에서 "중립"으로 조정한 경우가 실제로 있었고, 사후 검증에서 그 판단이 더 정확했다.

**한계**: LLM의 판단은 결정론적이지 않다. 같은 데이터를 넣어도 다른 판단이 나올 수 있다. `temperature=0`이어도 완전히 동일하지는 않다.

LLM이 사후적 합리화를 하는지, 실제 데이터 기반 추론인지 구분하기 어렵다. `daily_verify` DAG로 사후 검증하는 것이 현재로서는 최선이다.

### 다시 설계한다면

- **클라우드 전환**: 로컬 환경의 한계(리소스, 안정성, 백업)를 체감했다. AWS에서 운영한다면 MWAA(Managed Workflows for Apache Airflow)를 검토할 것이다.
- **Dagster 재검토**: Airflow의 "Task 중심" 패러다임 대신 Dagster의 "Asset 중심" 패러다임이 이 프로젝트에는 더 적합했을 수 있다. "stock_prices 테이블이 최신 상태인가"를 기준으로 파이프라인이 동작하는 모델이 데이터 엔지니어링에 더 자연스럽다.
- **실시간 수집 검토**: 현재는 장 마감 후 배치 수집이지만, 장중 실시간 시세를 스트리밍으로 수집하면 더 정교한 판단이 가능할 것이다.

### 성장

크롤링과 백엔드 개발은 "데이터를 가져오고 처리하는 기술"이었다. 이 프로젝트는 "그걸 시스템으로 만드는 기술"이었다. 스크립트를 짜는 것과 파이프라인을 설계·운영하는 것은 근본적으로 다른 역량이다. 이 프로젝트를 통해 수집 이후의 세계 — 정제, 적재, 자동화, 모니터링, 장애 대응 — 에 눈을 뜨게 되었다.

---

전체 코드는 GitHub에서 확인할 수 있다: [GitHub 저장소](https://github.com/djgnfj-svg/trader_ai)
