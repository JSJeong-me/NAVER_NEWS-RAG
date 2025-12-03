1단계 설계를 바탕으로 '2 단계: 1단계에서 작성한 'process 중심의 설계'를 구현하는 함수와 data type의 signature 중심의 설계' 를 작성해 주세요. 이떄로 설계에만 집중하고 code 는 작성하지 마세요.



좋아요, 이제 1단계의 프로세스 설계를 바탕으로 **“무슨 함수와 데이터 타입을 어떻게 나눌지”**를 정리하는 2단계로 갈게요.
여전히 **설계만** 하고, 실제 코드/구현은 안 씁니다.

---

## 1. 전체 패키지 / 모듈 구조 (설계안)

`/home/ubuntu/news_regime` 안에 다음과 같은 논리적 구조를 둔다고 가정합니다.

* `analyst_agent/`

  * `config.py`          : 설정 관련 타입 및 로더
  * `models.py`          : 도메인 데이터 타입(뉴스, 분석, Regime, Report 등)
  * `cli_tools.py`       : `run_crawler.sh`, `query_db.sh`, `list_news.sh` 래퍼
  * `parse_query_output.py` : `query_db.sh all` 출력 파서
  * `analysis_logic.py`  : Regime/변동성/추천/심리 집중 분석 로직
  * `report_schema.py`   : AnalystReport 구조 정의 (Trader Agent와 공유)
  * `report_generator.py`: LLM(Agents SDK) 기반 리포트 생성
  * `orchestrator.py`    : run_once 사이클 orchestration
  * `agent_definitions.py`: OpenAI Agents SDK용 Analyst Agent 정의
  * `run_once.py`        : `python -m analyst_agent.run_once` 엔트리포인트

이 구조를 기준으로, 각 모듈에 들어갈 **데이터 타입(Data Types)** 과 **함수(Functions) 시그니처**를 정리합니다.

---

## 2. 설정 / Config 관련 설계 (`config.py`)

### 2.1 데이터 타입

* `AnalystAgentConfig`

  * 필드:

    * `hours_window: int`                  (분석 대상 시간창, 기본 24)
    * `news_limit: int`                    (뉴스 최대 개수, 기본 50)
    * `run_crawler_before_analysis: bool`  (분석 전에 크롤러 실행 여부)
    * `report_output_dir: str`             (리포트 파일 디렉토리)
    * `report_format: str`                 (예: "markdown", "text")
    * `log_dir: str`
    * `regime_window_size: int`            (Regime 변화 감지에 고려할 최근 항목 수)
    * `volatility_thresholds: VolatilityThresholdConfig` (임팩트/리스크 기준 설정)
    * `llm: LlmConfig`                     (LLM 모델/temperature 등)
    * `scheduler_mode: str`                (예: "once", "loop", "external_cron")
    * `env_name: str`                      (예: "prod", "dev")

* `VolatilityThresholdConfig`

  * 필드 예:

    * `impact_normal_max: float`
    * `impact_caution_min: float`
    * `impact_spike_min: float`
    * `risk_flag_caution_ratio: float`
    * `risk_flag_spike_ratio: float`

* `LlmConfig`

  * 필드:

    * `model: str`
    * `temperature: float`
    * `max_tokens: int`
    * `request_timeout: float`
    * (옵션) `response_format: str` (예: "text", "json")

### 2.2 주요 함수 시그니처

* `load_analyst_agent_config() -> AnalystAgentConfig`

  * 설명: 환경변수/설정 파일을 읽어 `AnalystAgentConfig` 생성.

---

## 3. 도메인 모델 설계 (`models.py` + `report_schema.py`)

### 3.1 DB 스냅샷 관련 타입 (`models.py`)

* `NewsArticle`

  * `index: int`                 (표시용 번호)
  * `title: str`
  * `publisher: str`
  * `category: str`
  * `crawled_at: datetime`
  * `url: str`
  * `summary_raw: str`           (원문 summary 라인)
  * `summary_clean: str`         (전처리/정제된 summary)

* `ArticleStats`

  * `total_articles: int`
  * `avg_sentiment: float`
  * `avg_impact: float`
  * `regulatory_risk_flags: int`
  * `earnings_risk_flags: int`
  * `credit_risk_flags: int`

* `SymbolStats`

  * `symbol: str`
  * `article_count: int`
  * `avg_score: float`

* `RecommendationStatus` (열거형/enum 개념)

  * 값 예: `"ACTIVE"`, `"INACTIVE"`, `"HOLD"`, …

* `Recommendation`

  * `symbol: str`
  * `status: RecommendationStatus`
  * `created_at: datetime`
  * `sentiment: float`
  * `impact: float`
  * `composite: float`
  * `risk_ratios: dict[str, float]`  (예: `"regulatory_risk_ratio": 0.0` 등)
  * `reason: str`                    (내부 룰베이스 설명)

* `RegimeLabel` (열거형 개념)

  * 값 예: `"risk_on"`, `"risk_off"`, `"sideways"`, `"unknown"`

* `RegimeRecord`

  * `timestamp: datetime`
  * `label: RegimeLabel`
  * `confidence: float`          (0~1 또는 0~100%)
  * `description: str`           (요약 코멘트)

* `DbSnapshot`

  * `generated_at: datetime`     (query_db 실행 시각)
  * `hours_window: int`
  * `news: list[NewsArticle]`
  * `article_stats: ArticleStats`
  * `symbol_stats: list[SymbolStats]`
  * `recommendations: list[Recommendation]`
  * `regime_history: list[RegimeRecord]`

### 3.2 분석 결과 관련 타입 (`analysis_logic.py`, `report_schema.py`)

* `RegimeChange`

  * `has_changed: bool`
  * `previous: RegimeRecord | None`
  * `current: RegimeRecord | None`
  * `change_direction: str | None`
    (예: `"risk_on_to_sideways"`, `"sideways_to_risk_off"`, `"none"`)

* `VolatilityLevel` (열거형 개념)

  * 값 예: `"NORMAL"`, `"WATCH"`, `"CAUTION"`, `"SPIKE"`

* `VolatilityAssessment`

  * `level: VolatilityLevel`
  * `reason: str`               (왜 그 레벨인지 요약)
  * `key_drivers: list[str]`    (변동성을 키우는 주요 요인 리스트)

* `FocusSymbol`

  * `symbol: str`
  * `article_count: int`
  * `avg_sentiment: float`
  * `is_recommended: bool`
  * `notes: str`                (요약 코멘트)

* `KeyNewsDriver`

  * `title: str`
  * `summary: str`
  * `impact_direction: str`     (예: `"positive"`, `"negative"`, `"mixed"`)
  * `impact_scope: str`         (예: `"market"`, `"sector"`, `"stock"`)
  * `reason: str`               (중요한 드라이버로 선택된 이유)

* `AnalystCoreFindings`

  * `current_regime: RegimeRecord | None`
  * `regime_change: RegimeChange`
  * `volatility: VolatilityAssessment`
  * `top_recommendations: list[Recommendation]`
  * `focus_symbols: list[FocusSymbol]`
  * `key_news_drivers: list[KeyNewsDriver]`
  * `article_stats: ArticleStats`
  * `raw_snapshot: DbSnapshot`  (필요시 원 데이터도 포함)

### 3.3 최종 리포트 타입 (`report_schema.py`)

* `AnalystReport`

  * `generated_at: datetime`
  * `config: AnalystAgentConfig` (또는 핵심 필드 subset)
  * `core_findings: AnalystCoreFindings`
  * `formatted_text: str`       (LLM이 생성한 최종 리포트 텍스트)
  * (옵션) `summary_header: str` (한 줄 요약)
  * (옵션) `metadata: dict[str, Any]`

이 `AnalystReport`가 **향후 Trader Agent의 입력**이 되는 공식 인터페이스입니다.

---

## 4. CLI 래퍼 설계 (`cli_tools.py`)

### 4.1 보조 타입

* `CliCommandResult`

  * `command: list[str]`       (실행된 명령어 전체)
  * `stdout: str`
  * `stderr: str`
  * `exit_code: int`
  * `started_at: datetime`
  * `finished_at: datetime`
  * `success: bool`

### 4.2 함수 시그니처

* `run_crawler_command(timeout_seconds: int | None) -> CliCommandResult`

  * 역할: `./run_crawler.sh` 실행.
  * 실패 시 `success=False`, stderr 포함.

* `run_query_db_all(hours: int, limit: int, timeout_seconds: int | None) -> CliCommandResult`

  * 역할: `./query_db.sh all --hours {hours} --limit {limit}` 실행.

* (선택) `run_query_db_news(...)`, `run_query_db_regime(...)` 등 세부 서브커맨드용.

* (선택) `run_list_news(timeout_seconds: int | None) -> CliCommandResult`

---

## 5. 파서 설계 (`parse_query_output.py`)

### 5.1 함수 시그니처

* `split_query_all_output(raw_output: str) -> dict[str, str]`

  * 반환: 섹션 이름 → 해당 섹션 텍스트
    예: `"news_articles"`, `"article_analysis"`, `"symbol_mappings"`, `"recommendations"`, `"regime_history"`.

* `parse_news_section(section_text: str) -> list[NewsArticle]`

* `parse_article_analysis_section(section_text: str) -> ArticleStats`

* `parse_symbol_mappings_section(section_text: str) -> list[SymbolStats]`

* `parse_recommendations_section(section_text: str) -> list[Recommendation]`

* `parse_regime_history_section(section_text: str) -> list[RegimeRecord]`

* `build_db_snapshot(raw_output: str, hours_window: int) -> DbSnapshot`

  * 내부적으로 위 파서들을 호출하여 `DbSnapshot` 구성.

---

## 6. 분석 로직 설계 (`analysis_logic.py`)

### 6.1 Regime/변동성/종목 분석 관련 함수

* `analyze_regime_change(regime_history: list[RegimeRecord], window_size: int) -> RegimeChange`

  * 최신/직전 Regime 비교 및 변화 방향 판단.

* `assess_volatility(article_stats: ArticleStats, regime_history: list[RegimeRecord], thresholds: VolatilityThresholdConfig) -> VolatilityAssessment`

  * Impact, risk flags, regime 변화를 조합해 변동성 레벨 결정.

* `select_top_recommendations(recommendations: list[Recommendation], max_count: int) -> list[Recommendation]`

  * composite, 최근 시각 등을 기준으로 상위 추천 종목 선택.

* `identify_focus_symbols(symbol_stats: list[SymbolStats], recommendations: list[Recommendation], max_count: int) -> list[FocusSymbol]`

  * 기사 수/avg_score/추천 여부 기반으로 심리 집중 종목 선정.

* `extract_key_news_drivers(news: list[NewsArticle], article_stats: ArticleStats, regime_history: list[RegimeRecord], max_count: int) -> list[KeyNewsDriver]`

  * 정책/세제/AI/사고 이슈 등 중요한 기사들을 heuristics로 선택.

* `build_core_findings(snapshot: DbSnapshot, config: AnalystAgentConfig) -> AnalystCoreFindings`

  * 위 함수들을 조합해 핵심 분석 결과를 하나의 구조로 묶음.

---

## 7. LLM / Agent 레이어 설계 (`agent_definitions.py`, `report_generator.py`)

### 7.1 LLM Client 래핑 (`llm_client` 개념)

* `LlmClientConfig` (이미 `LlmConfig`로 존재하므로 재사용 가능)
* `LlmClient`

  * 메서드 예:

    * `generate_report_text(core_findings: AnalystCoreFindings, config: AnalystAgentConfig) -> str`

      * 실제로는 Agents SDK 기반 Agent 호출을 내부에서 수행.

### 7.2 Agents SDK 기반 Analyst Agent (개념 설계)

* `create_analyst_agent(llm_config: LlmConfig) -> AnalystAgentHandle`

  * 반환: SDK가 제공하는 Agent 핸들/클라이언트 객체.
  * 내부: system prompt/role, 툴 접근 권한, 응답 형식 정의.

* `run_analyst_agent_report(agent: AnalystAgentHandle, core_findings: AnalystCoreFindings, config: AnalystAgentConfig) -> str`

  * Agent에 core_findings(또는 JSON)을 context로 보내고,
  * 정해진 리포트 포맷 지침에 따라 텍스트 리포트 생성.

※ 실제 구현에서는 `LlmClient.generate_report_text()`가 내부적으로 이 Agent를 사용하도록 연결할 수 있습니다.

### 7.3 리포트 생성 레벨 (`report_generator.py`)

* `generate_analyst_report(core_findings: AnalystCoreFindings, config: AnalystAgentConfig, llm_client: LlmClient) -> AnalystReport`

  * 내부:

    * Analyst Agent를 호출하여 `formatted_text` 생성.
    * `AnalystReport` 객체를 구성하여 반환.

---

## 8. 오케스트레이션 / 엔트리포인트 설계 (`orchestrator.py`, `run_once.py`)

### 8.1 실행 사이클 함수

* `run_analyst_cycle_once(config: AnalystAgentConfig) -> AnalystReport`

  * 단계:

    * 필요 시 `run_crawler_command`
    * `run_query_db_all`
    * `build_db_snapshot`
    * `build_core_findings`
    * `generate_analyst_report`
    * 리포트 파일 저장/로그 기록 (또는 별도 함수에 위임)

* `save_analyst_report(report: AnalystReport) -> str`

  * 리포트 파일 경로를 반환.

* `log_analyst_cycle(report: AnalystReport, cli_results: dict[str, CliCommandResult]) -> None`

### 8.2 모듈 엔트리포인트 (`run_once.py`)

* `main() -> None`

  * `AnalystAgentConfig` 로딩
  * `run_analyst_cycle_once` 호출
  * stdout에 리포트 출력/종료

(실제 실행은 `python -m analyst_agent.run_once` 형태)

---

## 9. Trader Agent 확장 포인트에 대한 인터페이스 준비

Trader Agent 연동을 대비해, 아래 인터페이스를 미리 고정하는 것이 중요합니다.

* `AnalystReport` 구조 (특히 `core_findings` 필드)
* `AnalystReport`를 JSON으로 직렬화/역직렬화하는 함수:

  * 예: `serialize_analyst_report(report: AnalystReport) -> dict`
  * 예: `deserialize_analyst_report(data: dict) -> AnalystReport`

Trader Agent는 나중에:

* `load_latest_analyst_report() -> AnalystReport`
* 또는 파일 경로/REST 응답 등을 통해 같은 구조를 받는 것을 전제로 설계할 수 있습니다.

---

여기까지가 2단계, 즉:

> 1단계 프로세스 설계를 실제로 구현하기 위해
> “어떤 데이터 타입이 필요하고, 어떤 함수들이 어떤 인자와 반환값을 가져야 하는지”를
> 중심으로 정리한 설계안입니다.

다음 3단계에서는 이 시그니처들을 바탕으로:

* 각 함수 안에서 어떤 순서로 무엇을 할지,
* 변수/중간 단계에 어떤 이름/의미를 줄지,

를 **코드 없이, 상세한 주석/의사코드 수준**으로 풀어서 설계하면 됩니다.
