agent framework 는 'OpenAI Agents SDK'를 사용하겠습니다.
지금부터 'OpenAI Agents SDK'를 사용하여 위에 정의된 '요구사항'을 프로그램으로 구현하는 '설계'를 진행합니다.
당신의 context window 를 고려하여 설계는 3단계로 진행 하겠습니다.
1 단계: '요구사항'을 구현하는 scenario 기반의 workflow 를 반영한 process 중심의 설계
2 단계: 1단계에서 작성한 'process 중심의 설계'를 구현하는 함수와 data type의 signature 중심의 설계
3 단계: 함수내의 code 가 없는 수준의 pseudo code 로 각 함수와 선언 변수에 '상세한 comments'를 작성 단계로 설계를 진행합니다.
그럼 위 3단계를 이해하고 '1단계' 를 진행해 주세요. 이때 '설계'에만 집중하고 code 는 작성하지 마세요.


먼저 단계 정리 잘 해주셔서, 그 틀에 맞춰갈게요.
지금은 **1단계: 시나리오 기반, 프로세스 중심 설계**만 합니다. (코드·시그니처 없음)

---

## 0. 전체 그림: 우리가 만들 Analyst Agent의 역할

* 실행 환경: `/home/ubuntu/news_regime` 디렉터리
* 주기: 기본 30분(설정 가능), **각 실행은 하나의 “리포트 생성 사이클”**
* 사용 도구:

  * `run_crawler.sh` (선택적 – 최신 데이터 확보)
  * `query_db.sh all --hours H --limit N` (핵심)
* 주요 단계:

  1. (옵션) 최신 뉴스/분석으로 DB 업데이트
  2. DB에서 지난 24시간(또는 설정된 기간) 데이터 조회
  3. 텍스트 출력 파싱 → 내부 구조화 데이터로 변환
  4. Regime 변화/변동성 판단 + 종목 추천/핵심 종목/뉴스 드라이버 추출
  5. Analyst Agent가 최종 리포트 텍스트 생성
  6. 리포트 저장 + 로그 남기기
* OpenAI Agents SDK:

  * “Analyst Agent”는 SDK의 Agent로 정의
  * CLI 호출/파싱/분석은 대부분 “Tool” + 일부 Python 로직으로 구현
  * **향후 Trader Agent**가 Analyst 결과를 그대로 받아서 사용할 수 있도록, **중간 결과(AnalystReport)를 구조화된 데이터로도 유지**

---

## 1. 실행 시나리오별 상위 프로세스

### 1-1. 배치 실행(30분마다) – 기본 시나리오

외부에서:

```bash
cd /home/ubuntu/news_regime
python -m analyst_agent.run_once
```

혹은 cron/systemd가 30분마다 수행.

이 때 내부 **단일 사이클 프로세스**:

1. **환경/설정 로딩**

   * Config 파일/환경변수에서:

     * 분석 시간창: `hours_window` (기본 24h)
     * 뉴스 limit: `news_limit` (기본 50)
     * Regime 변화 민감도, Volatility 레벨 기준 등
     * `run_crawler_before_analysis` (True/False)
   * 로그/리포트 디렉토리 경로 로딩.

2. **(옵션) 크롤러 실행 단계**

   * 설정이 `run_crawler_before_analysis=True` 이면:

     * `run_crawler.sh`를 호출하는 Tool 실행.
     * 성공/실패 여부, 실행 시간 로깅.
   * 실패 시:

     * 이번 사이클은 **기존 DB 데이터만으로 분석**, 리포트에 “데이터 업데이트 실패” 주석 추가.

3. **DB 조회 단계**

   * 핵심 Tool:

     * `query_db.sh all --hours {H} --limit {N}`
   * 표준 출력(stdout)을 캡처:

     * News Articles 섹션
     * Article Analysis 섹션
     * Symbol Mappings 섹션
     * Recent Recommendations 섹션
     * Regime History 섹션
   * 오류/timeout 발생 시:

     * 재시도 1~2회 후 실패 시, “이번 사이클에서는 분석 데이터 확보 실패” 리포트.

4. **출력 파싱/데이터 구조화 단계**

   * Python 로직 또는 별도 Tool로:

     * `query_db.sh` 전체 결과를 섹션별 텍스트로 분리.
     * 각 섹션을 도메인 모델로 변환:

       * NewsArticle 목록 (제목/요약/시간/URL 등)
       * ArticleStats (뉴스 개수, 평균 감성/임팩트, 리스크 플래그 수)
       * SymbolStats (각 종목별 기사 수, 평균 score)
       * Recommendations (종목, composite score, reason 등)
       * RegimeHistory (timestamp, label, confidence, 설명)
   * 파싱 실패 시:

     * 해당 섹션만 None/Empty로 처리하고, 리포트에 “부분 데이터 누락” 표시.

5. **분석/결론 도출 단계**

   * **Regime 분석**

     * RegimeHistory에서:

       * 최신 Regime: label + confidence
       * 직전 Regime과 비교 → Regime Change 여부/방향
       * 최근 K개(예: 3개) Regime의 변화 패턴 분석.
   * **변동성·리스크 레이더 분석**

     * ArticleStats 기반:

       * 평균 Impact 수준이 높아졌는지
       * Earnings/Credit/Regulation 등 리스크 플래그 비율
     * Regime 변화와 함께 종합적으로:

       * Volatility 상태를 NORMAL / WATCH / CAUTION / SPIKE 중 하나로 분류.
   * **추천 종목 분석**

     * Recommendations 데이터를 정렬/필터:

       * 상위 N개 종목(예: 5개)의 상태/점수/리스크 구조화.
       * 각 종목에 대해:

         * 심리 지표, 관련 기사 수, risk ratio 요약.
     * 현재 Regime과 조합:

       * Regime이 risk_on/sideways/risk_off 일 때 해당 종목이 어느 정도 정합적인지 논리 판단.
   * **심리 집중 종목 분석**

     * SymbolStats에서 기사 수 상위 종목, 높은 sentiment 종목 추출.
   * **대표 뉴스 드라이버 선정**

     * NewsArticles 중:

       * 정책/세제, 대형 수혜, 사건사고 등 “시장 전체/섹터/핵심 종목에 영향 큰” 기사 몇 개를 선택.
     * 각 기사에 대해:

       * 핵심 메시지, 잠재 영향 방향/강도 추출.

6. **LLM 기반 리포트 생성 단계 (Analyst Agent의 핵심)**

   * OpenAI Agents SDK의 Analyst Agent가:

     * 위에서 구조화된 데이터(정리된 JSON)를 **context로 입력** 받아
     * 우리가 정의한 리포트 포맷:

       * [Regime Alert]
       * [Market Overview]
       * [Volatility & Risk Radar]
       * [Top Recommendations]
       * [Focus Symbols]
       * [Key News Drivers]
       * [Summary & Checkpoints]
     * 에 맞춰 자연어 보고서를 생성.
   * 이 과정에서:

     * LLM은 **Regime 변화·변동성·추천 종목·리스크**에 대한 “설명/코멘트”에 집중.
     * 정량 데이터/핵심 지표는 이미 계산된 것을 깔끔하게 텍스트로 풀어주는 역할.

7. **리포트 출력/저장 단계**

   * 생성된 리포트를:

     * **stdout**에 출력 (터미널/로그에서 바로 확인 가능).
     * 파일로 저장:

       * 예: `/home/ubuntu/news_regime/reports/analyst_report_YYYYMMDD_HHMM_<regime>.md`
     * (선택) 구조화 JSON도 함께 저장:

       * 향후 Trader Agent/리스크 관리 시스템에서 재사용.

8. **로그/헬스체크**

   * 각 단계의:

     * 시작/종료 시간
     * 크롤러 호출 성공 여부
     * query_db 콜 성공 여부
     * 파싱 성공 여부
     * 최종 Regime + Volatility 상태
   * 를 로그 파일에 기록.
   * 나중에 “Analyst Agent가 제때 잘 돌아갔는지” 모니터링에 활용.

---

### 1-2. 대화형/수동 실행 시나리오 (향후를 위한 그림)

향후, 콘솔이나 웹 UI에서:

* “지금 즉시 Analyst 리포트 만들어줘”
* “지난 48시간 기준으로 다시 분석해줘”
* “Regime 변화가 있었던 마지막 시점만 요약해줘”

같은 요청을 하고 싶은 상황을 고려하면:

1. 사용자가 파라미터 입력:

   * 분석 시간창(`hours_window=48`)
   * 뉴스 limit (`limit=20`)
   * 리포트 타입 (full / short / regime-only)

2. Orchestrator 또는 간단한 CLI가:

   * 해당 파라미터로 Analyst Agent 실행 (위의 동일 파이프라인, 단 한번만 실행).
   * 결과 리포트를 화면/웹으로 반환.

이 흐름은 **배치 실행과 거의 동일**하고, 단지 외부 트리거가 cron이 아니라 사용자의 요청이라는 차이만 있음.
그래서 설계에서 “단일 사이클 run_once 프로세스”를 기본 유닛으로 잡는 게 중요합니다.

---

### 1-3. 향후 Trader Agent 연동을 고려한 프로세스 연결

지금 단계에서 직접 Trader Agent를 구현하지는 않지만, **프로세스 설계에 미리 연결 지점을 만들어 둡니다.**

#### (A) Analyst → Trader 데이터 전달 포인트

위 사이클 6~7단계 즈음에서:

* 구조화된 Analyst 결과 객체(예: `AnalystReport`)를 만든다:

  * `current_regime`, `prev_regime`, `regime_change`, `volatility_level`
  * `top_recommendations` (종목, 점수, 리스크 비율 등)
  * `focus_symbols` (뉴스/심리가 집중된 종목)
  * `key_risk_flags` (regulation/earnings/credit 등)
* 이 객체를:

  * 파일(JSON)로 저장: `/reports/analyst_report_...json`
  * 또는 향후 Python API를 통해 바로 Trader Agent에 전달할 수 있게 설계.

#### (B) Trader Agent 실행 시나리오 (미래 대비)

향후 Trader Agent 사이클:

1. Trader Agent 실행:

   * 최근 AnalystReport JSON을 로드.
   * 계좌/포지션 상태, 종목별 가격/수급 데이터 등을 추가로 로드.

2. Trader Agent 분석:

   * Regime/Volatility/추천 종목/집중 종목/리스크 플래그를 참고해서:

     * 오늘/이번 시점에 어떤 종목을 Long/Short/No-Trade 후보로 볼지 판단.
   * 필요한 경우:

     * Analyst Agent에게 “특정 종목/섹터에 대해서만 추가 뉴스 요약”을 재요청하는 루프도 가능 (멀티에이전트 대화).

이렇게 보면 현재 **Analyst Agent의 핵심 산출물(AnalystReport)**이,
나중에 **Trader Agent의 공식 입력 인터페이스** 역할을 하므로,
1단계 설계에서 “AnalystReport 구조”를 잘 정의해 두는 게 매우 중요합니다.

---

## 2. OpenAI Agents SDK 관점에서의 논리적 구성

(여전히 코드 없이, 구조/프로세스만)

### 2-1. 툴 레벨 프로세스

* Tool 그룹 1: **External CLI Tools**

  * `tool.run_crawler()`

    * `run_crawler.sh` shell 실행
  * `tool.query_db_all(hours, limit)`

    * `query_db.sh all --hours {hours} --limit {limit}` 실행
    * stdout 텍스트 반환

* Tool 그룹 2: **Parsing / Domain 전처리 Tools**

  * `tool.parse_query_all_output(raw_text)`

    * 섹션별로 분리 + 도메인 구조화 (News/Analysis/Symbol/Recommendations/Regime).
  * (선택) `tool.summarize_news_for_agent` 등 LLM 보조 전처리.

* Tool 그룹 3: **System/Utility Tools**

  * `tool.get_current_time()`, `tool.write_report_file(content, meta)` 등.

*실제 구현에서는 일부를 순수 Python 로직으로 하고, Agent에는 “필요한 최소 Tool만 보이게” 할 수 있음.*

### 2-2. Agent 프로세스

* **Analyst Agent**

  * Role:

    * “뉴스 기반 Regime/변동성/추천 종목을 분석하여 리포트를 작성하는 전문가 애널리스트”
  * 입력:

    * 파싱/전처리까지 마친 구조화 데이터 (또는 일부 Text + 구조화 JSON)
  * 가능한 행동:

    * 툴 호출 (예: query_db_all, parse_output) – 구현 방식에 따라,

      * Orchestrator가 대부분을 수행하고 Agent는 리포트 작성에만 집중할 수도 있고,
      * Agent에게 `query_db` 툴을 직접 넘겨 “필요할 때만 호출”하게 할 수도 있음.
  * 출력:

    * 최종 리포트 텍스트
    * (옵션) 정리된 요약 JSON (요약 헤더/체크포인트 등)

* **Orchestrator (Python 코드, SDK 외부 레벨)**

  * 전체 사이클의 프로세스를 통제:

    * 설정 로딩
    * (옵션) run_crawler 툴 호출
    * query_db_all 툴 호출 & 결과 파싱
    * 분석/지표 계산 (필요시 Python 함수)
    * Analyst Agent에게 “리포트 생성” 요청
    * 결과 저장/출력
  * 향후:

    * Trader Agent까지 포함하는 상위 orchestrator로 확장.

---

## 3. 1단계 정리 – 우리가 갖게 될 “프로세스 청사진”

요약하면, 1단계에서 정리된 **프로세스 중심 설계**는 다음과 같습니다:

1. **run_once 사이클 정의**

   * Config 로딩 → (옵션 run_crawler) → query_db_all → 파싱 → 분석/지표 계산 → Analyst Agent 리포트 → 저장/출력 → 로그.

2. **Analyst Agent 역할**

   * 파싱·분석된 구조화 데이터를 받아,
   * Regime 변화/변동성/추천/심리 집중/대표 뉴스 드라이버를 자연어로 엮어,
   * 트레이더가 바로 참고할 수 있는 표준 리포트 포맷으로 출력.

3. **Trader Agent 확장 포인트**

   * AnalystReport를 구조화된 형태(JSON)로 저장/전달 가능하게 설계.
   * 이후 Trader Agent가 이를 입력으로 받아 추가 판단/전략 제안.

4. **OpenAI Agents SDK 사용 방식**

   * CLI 호출/파싱/저장 등은 대부분 Python 함수/Tool로 구현.
   * Analyst Agent는:

     * 주로 리포트 생성(요약/설명)에 집중,
     * 필요시 일부 툴을 직접 호출할 수도 있는 구조.

---

다음 단계(2단계)에서는 이 프로세스를 기반으로:

* 어떤 **Python 모듈/클래스/함수**를 만들지,
* 각 함수/데이터 타입이 어떤 인자와 반환값을 가지는지 (signature 차원),
* `AnalystReport` 같은 핵심 데이터 구조를 어떻게 정의할지

를 구체적으로 설계해 보겠습니다.
