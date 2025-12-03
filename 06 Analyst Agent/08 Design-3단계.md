다음 3단계에서는 이 시그니처들을 바탕으로: 각 함수 안에서 어떤 순서로 무엇을 할지, 변수/중간 단계에 어떤 이름/의미를 줄지, 를 코드 없이, 상세한 주석/의사코드 수준으로 풀어서 설계를 완성해 주세요.



좋아요, 이제 3단계입니다.
앞에서 설계한 함수/데이터 타입을 기준으로, **“각 함수 안에서 무슨 일을 어떤 순서로 할지”를 코드 없이 설명형/의사코드로 풀어** 볼게요.

너무 길어지지 않도록, **핵심 파이프라인에 직접 관여하는 함수들 위주**로 상세히 쓰고, 나머지 유사 패턴 함수는 짧게 정리하겠습니다.

---

# 1. 설정 / Config 로딩 (`config.py`)

### 1-1. `load_analyst_agent_config() -> AnalystAgentConfig`

**의사코드 / 순서**

1. 현재 프로세스의 환경변수와 설정 파일(예: `analyst_agent.yaml`)을 읽을 준비를 한다.
2. 설정 파일 위치 결정:

   * 기본 경로: `/home/ubuntu/news_regime/config/analyst_agent.yaml`
   * 환경변수 `ANALYST_CONFIG_PATH`가 있으면 그 경로를 우선 사용한다.
3. 설정 파일이 존재하면:

   * YAML/JSON 파서를 사용해 내용 읽기.
   * 키: `hours_window`, `news_limit`, `run_crawler_before_analysis`, `report_output_dir`, `log_dir` 등.
   * 값이 없으면 기본값으로 채운다.
     예: `hours_window` 없으면 24, `news_limit` 없으면 50 등.
4. 환경변수 override 처리:

   * 예: `ANALYST_HOURS_WINDOW`, `ANALYST_NEWS_LIMIT`, `ANALYST_RUN_CRAWLER` 등이 있으면,
   * 설정 파일에서 읽은 값 대신 환경변수 값을 우선 적용한다.
5. `VolatilityThresholdConfig` 생성:

   * 설정 파일/환경변수에서 impact 기준, risk_flag 기준 등을 읽어 구성한다.
6. `LlmConfig` 생성:

   * `LLM_MODEL`, `LLM_TEMPERATURE`, `LLM_MAX_TOKENS`, `LLM_TIMEOUT` 환경변수 또는 설정 파일에서 가져온다.
7. 위 정보들을 이용해 `AnalystAgentConfig` 인스턴스를 만든다.
8. 로그에 현재 설정 요약을 남긴다(예: `hours_window=24, news_limit=50, run_crawler_before_analysis=True`).
9. 구성된 `AnalystAgentConfig`를 반환한다.

---

# 2. CLI 래퍼 (`cli_tools.py`)

### 2-1. `run_crawler_command(timeout_seconds: int | None) -> CliCommandResult`

**의사코드 / 순서**

1. 실행할 명령어 리스트를 준비한다:

   * 예: `["./run_crawler.sh"]`
2. 현재 시각을 `started_at` 변수로 저장한다.
3. subprocess/프로세스 실행 유틸리티를 호출하여:

   * 위 명령어를 실행.
   * 지정된 `timeout_seconds` 안에 완료되도록 대기.
   * stdout, stderr, exit code를 수집한다.
4. 타임아웃 발생 시:

   * 프로세스를 종료/kill 처리.
   * exit code를 비정상 값(예: -1)로 설정.
   * stderr에 타임아웃 메시지를 추가한다.
5. 현재 시각을 `finished_at` 변수로 저장한다.
6. exit code가 0이면 `success=True`, 그 외는 `success=False`로 설정한다.
7. `CliCommandResult` 객체를 구성:

   * `command` = 실제 실행한 명령 리스트
   * `stdout`, `stderr`, `exit_code`, `started_at`, `finished_at`, `success`
8. 로그에 결과(성공/실패, 실행시간)를 남긴다.
9. `CliCommandResult`를 반환한다.

---

### 2-2. `run_query_db_all(hours: int, limit: int, timeout_seconds: int | None) -> CliCommandResult`

**의사코드 / 순서**

1. 인자로 받은 `hours`, `limit`을 문자열로 변환하여, 실행할 명령을 구성:

   * 예: `["./query_db.sh", "all", "--hours", str(hours), "--limit", str(limit)]`
2. 이 명령어 리스트를 가지고, 위 `run_crawler_command`와 동일한 패턴으로:

   * `started_at` 기록
   * 프로세스 실행
   * stdout/stderr/exit_code 수집
   * 타임아웃 처리
   * `finished_at` 기록
3. exit code 확인:

   * 0이면 `success=True`
   * 아니면 `success=False`, stderr를 통해 원인 파악 가능하게.
4. `CliCommandResult` 인스턴스를 생성하여 반환한다.
5. 로그에 “query_db all 호출 결과”를 요약 출력한다 (총 글자 수, 성공 여부 등).

---

# 3. 파싱 / 스냅샷 구축 (`parse_query_output.py`)

### 3-1. `split_query_all_output(raw_output: str) -> dict[str, str]`

**의사코드 / 순서**

1. `raw_output`을 줄 단위로 나눈다.
2. 섹션 구분 헤더 패턴을 미리 정의한다:

   * 예: `"News Articles ("` 로 시작하는 줄 → `news_articles` 섹션 시작
   * `"Article Analysis ("` → `article_analysis`
   * `"Symbol Mappings ("` → `symbol_mappings`
   * `"Recent Recommendations ("` → `recommendations`
   * `"Regime History ("` → `regime_history`
3. 현재 섹션 이름을 담는 변수 `current_section`을 `None`으로 초기화한다.
4. 섹션별 텍스트를 저장할 딕셔너리 `sections`를 `{"news_articles": "", ...}` 형식으로 초기화하거나, 필요 시 발견될 때 동적 생성한다.
5. 각 줄을 순회하면서:

   * 줄이 섹션 헤더 패턴과 매칭되면:

     * `current_section`을 해당 섹션 이름으로 변경.
     * 이 줄(헤더)도 섹션 텍스트에 포함할지 여부를 정책에 따라 결정.
   * `current_section`이 설정되어 있다면:

     * 해당 섹션의 텍스트에 현재 줄을 이어붙인다(개행 포함).
6. 모든 줄 처리 후 `sections`를 반환한다:

   * 키: `"news_articles"`, `"article_analysis"`, `"symbol_mappings"`, `"recommendations"`, `"regime_history"`
   * 값: 해당 섹션의 raw 텍스트(원래 형식 유지).

---

### 3-2. `parse_news_section(section_text: str) -> list[NewsArticle]`

**의사코드 / 순서**

1. `section_text`를 줄 단위로 분리한다.
2. 각 뉴스 항목은 예시처럼:

   * `번호. 제목` 줄
   * `Publisher: ...` 줄
   * `Category: ...` 줄
   * `Crawled: ...` 줄
   * `URL: ...` 줄
   * `Summary: ...` 줄
     의 패턴을 따름을 가정한다.
3. 반복 처리:

   * `번호. 제목` 패턴을 찾으면 **새 NewsArticle 블록 시작**:

     * 기존 블록이 있었다면 리스트에 추가.
     * 새 `current_article` 객체를 만들고:

       * `index` = 번호
       * `title` = 제목 부분 텍스트
   * `"Publisher:"`로 시작하는 줄에서 발행사 추출하여 `current_article.publisher`에 저장.
   * `"Category:"` 줄에서 카테고리 추출.
   * `"Crawled:"` 줄에서 datetime 문자열 파싱 → `current_article.crawled_at`.
   * `"URL:"` 줄에서 url 추출.
   * `"Summary:"` 줄에서 summary 원문 전체를 `summary_raw`에 저장.

     * 필요 시 이후에 간단 전처리(공백/특수문자 정리 등)를 수행하여 `summary_clean`에 저장.
4. 루프가 끝난 뒤 마지막 `current_article`도 리스트에 추가한다.
5. 결과 리스트를 반환한다.

---

### 3-3. `parse_article_analysis_section(section_text: str) -> ArticleStats`

**의사코드 / 순서**

1. `section_text`를 줄 단위로 순회한다.
2. `"Total Articles Analyzed:"` 포함 줄을 찾아 숫자를 추출 → `total_articles`.
3. `"Average Sentiment Score:"` 줄에서 float 값 추출 → `avg_sentiment`.
4. `"Average Impact Score:"` 줄에서 float 값 추출 → `avg_impact`.
5. `"Regulatory Risk Flags:"`, `"Earnings Risk Flags:"`, `"Credit Risk Flags:"` 줄에서 각각 정수 값을 추출한다.
6. `ArticleStats` 인스턴스를 생성해 위 값들을 설정한다.
7. `ArticleStats`를 반환한다.

---

### 3-4. `parse_symbol_mappings_section(section_text: str) -> list[SymbolStats]`

**의사코드 / 순서**

1. `"Symbols by Article Count:"` 이후의 줄들을 대상 구간으로 잡는다.
2. 각 줄은 예시처럼:

   * `005930: 5 articles (avg score: 0.580)`
3. 각 줄에 대해:

   * 기호 `:` 앞 부분 → `symbol`
   * `articles` 앞의 숫자 → `article_count`
   * `(avg score: X)` 안의 X 값을 float로 추출 → `avg_score`
4. `SymbolStats` 객체를 만들어 리스트에 추가.
5. 리스트를 반환한다.

---

### 3-5. `parse_recommendations_section(section_text: str) -> list[Recommendation]`

**의사코드 / 순서**

1. 섹션 텍스트를 줄 단위로 나눈다.
2. `"1. 035420 - ACTIVE"` 같은 라인을 기준으로 각 추천 블록 시작을 감지한다:

   * 숫자 + `.` + 종목코드 + `-` + 상태 패턴을 파싱.
3. 블록 안에서:

   * `"Created:"` 줄에서 datetime 문자열 추출 → `created_at`.
   * `"Sentiment: X | Impact: Y | Composite: Z"` 패턴에서 float 값들을 추출.
   * `"Risks: {...}"` 부분에서 dictionary 문자열을 파싱 → `risk_ratios`.
   * `"Reason: ..."` 줄 전체를 reason 문자열로 저장.
4. 각 블록마다 `Recommendation` 객체를 만들어 리스트에 추가.
5. 리스트 반환.

---

### 3-6. `parse_regime_history_section(section_text: str) -> list[RegimeRecord]`

**의사코드 / 순서**

1. 각 Regime 항목은 예시처럼:

   * `1. 2025-12-03 13:36:34.918289+09:00 - sideways (confidence: 60.00%)`
   * 다음 줄에 설명: `중립적 시장 심리`
2. 줄 단위로 순회하면서:

   * 번호 + 날짜 + `- label (confidence: X%)` 패턴을 발견하면 새 RegimeRecord 시작:

     * 날짜 → datetime 파싱.
     * label → RegimeLabel로 매핑 (risk_on, risk_off, sideways 등).
     * confidence 퍼센트 → float로 변환 (0~1 또는 0~100 중 사전에 결정).
   * 바로 다음 줄을 설명(description)으로 인식하여 `description` 필드에 저장.
3. 모든 RegimeRecord를 리스트에 모아 반환한다.

---

### 3-7. `build_db_snapshot(raw_output: str, hours_window: int) -> DbSnapshot`

**의사코드 / 순서**

1. `split_query_all_output(raw_output)`을 호출하여 섹션별 raw 텍스트 딕셔너리를 얻는다.
2. 각 섹션에 대해:

   * `news = parse_news_section(sections["news_articles"])`
   * `article_stats = parse_article_analysis_section(sections["article_analysis"])`
   * `symbol_stats = parse_symbol_mappings_section(sections["symbol_mappings"])`
   * `recommendations = parse_recommendations_section(sections["recommendations"])`
   * `regime_history = parse_regime_history_section(sections["regime_history"])`
3. 현재 시각을 `generated_at`으로 설정한다.
4. `DbSnapshot` 인스턴스를 생성:

   * 필드에 위 파싱 결과와 `hours_window`를 채운다.
5. `DbSnapshot`을 반환한다.

---

# 4. 분석 로직 (`analysis_logic.py`)

### 4-1. `analyze_regime_change(regime_history: list[RegimeRecord], window_size: int) -> RegimeChange`

**의사코드 / 순서**

1. `regime_history`를 `timestamp` 기준으로 오름차순 정렬한다.
2. 최신 항목을 `current`로, 직전 항목(있다면)을 `previous`로 설정한다.
3. `previous`가 없으면:

   * `has_changed=False`
   * `change_direction="none"`
   * `RegimeChange`를 그대로 반환.
4. `previous.label`과 `current.label`을 비교:

   * 같으면:

     * `has_changed=False`
     * `change_direction="none"`
   * 다르면:

     * `has_changed=True`
     * `change_direction`을 `"previousLabel_to_currentLabel"` 형식 문자열로 설정
       (예: `"risk_on_to_sideways"`).
5. `RegimeChange` 객체를 구성하여 반환:

   * `has_changed`, `previous`, `current`, `change_direction`.

---

### 4-2. `assess_volatility(article_stats: ArticleStats, regime_history: list[RegimeRecord], thresholds: VolatilityThresholdConfig) -> VolatilityAssessment`

**의사코드 / 순서**

1. `article_stats.avg_impact`를 기준으로 기본 VolatilityLevel을 결정:

   * avg_impact ≤ `impact_normal_max` → 초기 상태 `NORMAL`
   * avg_impact ≥ `impact_spike_min` → 초기 상태 `SPIKE`
   * 그 사이면 `WATCH` 또는 `CAUTION` 등으로 중간 레벨 설정.
2. 리스크 플래그 비율 계산:

   * 총 articles = `article_stats.total_articles` (0이면 리스크 플래그는 0으로 처리).
   * 각 리스크 플래그 수 / 총 기사 수를 통해 비율 계산.
   * 비율이 `risk_flag_spike_ratio` 이상이면 레벨을 상향 조정(SPIKE).
   * 비율이 `risk_flag_caution_ratio` 이상이면 최소 `CAUTION` 이상으로 조정.
3. Regime 변화도 반영:

   * `analyze_regime_change`를 호출해 최신/직전 Regime 분석.
   * 예:

     * risk_on → risk_off 같은 급격한 변화가 있으면 레벨을 한 단계 이상 상향.
     * sideways → risk_on처럼 변동성이 줄어드는 방향이면 레벨을 완화할 수도 있음(설정 정책에 따라).
4. 최종 결정된 `VolatilityLevel`을 `level` 필드에 저장.
5. `reason` 문자열을 구성:

   * 예: “평균 임팩트 0.155, 리스크 플래그 비율 낮음, 최근 Regime은 risk_on에서 sideways로 완화 → 변동성 경계 단계.”
6. `key_drivers` 리스트에는:

   * 변동성에 영향을 주는 주요 요소(impact, risk flags, Regime 변화 방향 등)를 간략히 텍스트로 나열.
7. `VolatilityAssessment`를 구성해 반환한다.

---

### 4-3. `select_top_recommendations(recommendations, max_count) -> list[Recommendation]`

**의사코드 / 순서**

1. `recommendations` 리스트에서 상태가 `ACTIVE`인 것만 필터링한다.
2. 필터된 리스트를 composite score 기준으로 내림차순 정렬한다.
3. 상위 `max_count` 개를 슬라이스해서 반환한다.
4. 추천이 없을 경우 빈 리스트 반환.

---

### 4-4. `identify_focus_symbols(symbol_stats, recommendations, max_count) -> list[FocusSymbol]`

**의사코드 / 순서**

1. 추천 종목의 symbol 집합을 미리 만든다:

   * 예: `recommended_symbols = {rec.symbol for rec in recommendations}`
2. `symbol_stats` 리스트를 기사 수(`article_count`) 기준으로 내림차순 정렬한다.
3. 상위 `max_count` 개를 순회하면서:

   * 각 심볼에 대해:

     * `is_recommended` = symbol이 `recommended_symbols`에 포함되는지 여부.
     * `notes` 텍스트:

       * 기사 수, 평균 sentiment, 추천 여부 등을 기반으로 간단 요약 작성.
   * `FocusSymbol` 객체를 생성해 리스트에 추가.
4. 리스트를 반환한다.

---

### 4-5. `extract_key_news_drivers(news, article_stats, regime_history, max_count) -> list[KeyNewsDriver]`

**의사코드 / 순서**

1. 뉴스 리스트에서 “시장/섹터 전체에 영향을 주는 이슈”를 heuristic으로 필터링:

   * 제목/summary에 특정 키워드 포함:

     * “거래세 인상”, “코스피”, “코스닥”, “정보유출”, “AI 특수”, “관세”, “규제”, …
   * 영향 범위(scope)를:

     * 지수/시장 → `"market"`
     * 특정 섹터(2차전지, 반도체 등) → `"sector"`
     * 특정 종목/사건 → `"stock"`
2. 후보 뉴스들 중에서:

   * 시간/임팩트(간접 추정) 등을 고려해 중요도를 점수화.
   * 상위 `max_count`개를 선택.
3. 각 뉴스에 대해 `KeyNewsDriver` 생성:

   * `title` = 뉴스 제목
   * `summary` = summary_clean 또는 summary_raw 정제판
   * `impact_direction` = 키워드/문맥을 기반으로 긍정/부정/혼합 추정
   * `impact_scope` = 위에서 판단된 `"market"`, `"sector"`, `"stock"`
   * `reason` = “거래세 인상으로 고회전 종목군 변동성 확대 가능” 등 설명.
4. 리스트 반환.

---

### 4-6. `build_core_findings(snapshot: DbSnapshot, config: AnalystAgentConfig) -> AnalystCoreFindings`

**의사코드 / 순서**

1. Regime 관련 분석:

   * `regime_change = analyze_regime_change(snapshot.regime_history, config.regime_window_size)`
   * `current_regime = regime_change.current`
2. 변동성 분석:

   * `volatility = assess_volatility(snapshot.article_stats, snapshot.regime_history, config.volatility_thresholds)`
3. 추천 종목:

   * `top_recs = select_top_recommendations(snapshot.recommendations, max_count=5)` (또는 설정값)
4. 심리 집중 종목:

   * `focus_symbols = identify_focus_symbols(snapshot.symbol_stats, top_recs, max_count=3~5)`
5. 주요 뉴스 드라이버:

   * `key_drivers = extract_key_news_drivers(snapshot.news, snapshot.article_stats, snapshot.regime_history, max_count=5)`
6. `AnalystCoreFindings` 객체 생성:

   * `current_regime = current_regime`
   * `regime_change = regime_change`
   * `volatility = volatility`
   * `top_recommendations = top_recs`
   * `focus_symbols = focus_symbols`
   * `key_news_drivers = key_drivers`
   * `article_stats = snapshot.article_stats`
   * `raw_snapshot = snapshot`
7. `AnalystCoreFindings` 반환.

---

# 5. LLM / Agent 레이어 (`LlmClient`, `generate_analyst_report`)

### 5-1. `LlmClient.generate_report_text(core_findings, config) -> str`

**의사코드 / 순서**

1. `core_findings`와 `config`에서 리포트에 필요한 데이터만 추출해 요약 JSON을 준비:

   * 현재 Regime 라벨/신뢰도
   * Regime 변화 여부/방향
   * Volatility level + reason
   * 상위 추천 종목 리스트 (symbol, scores, reason)
   * Focus symbols
   * Key news drivers (title/summary/scope/direction)
   * ArticleStats (평균 감성/임팩트, 리스크 플래그)
2. 이 요약 JSON을 문자열 또는 LLM context로 사용할 수 있는 구조로 포맷한다.
3. 프롬프트 템플릿 구성:

   * system 역할:

     * “당신은 주식 시장 뉴스 기반 애널리스트입니다. 아래 JSON 데이터와 리포트 포맷 규칙을 바탕으로, 트레이더가 바로 참고할 수 있는 분석 리포트를 작성하세요. 투자 권유가 아니라 정보 분석임을 명시하세요.” 등의 지침.
   * user/assistant 역할:

     * 입력 JSON 포함
     * 출력은 지정된 섹션 구조:

       * `[Regime Alert]`, `[Market Overview]`, `[Volatility & Risk Radar]`, `[Top Recommendations]`, `[Focus Symbols]`, `[Key News Drivers]`, `[Summary & Checkpoints]`.
4. OpenAI Agents SDK를 사용해 Analyst Agent를 호출:

   * Agent가 tool을 사용할 필요가 거의 없다면, 순수 텍스트 생성만 수행하게 설정.
5. 응답으로부터 텍스트 리포트를 추출한다:

   * response 객체 내 최종 message/content를 문자열로 가져온다.
6. 필요 시, 리포트 텍스트에 간단한 post-processing:

   * 공백/줄바꿈 정리
   * 헤더/섹션 제목 마크다운 정렬 등.
7. 최종 텍스트 문자열을 반환한다.

---

### 5-2. `generate_analyst_report(core_findings, config, llm_client) -> AnalystReport`

**의사코드 / 순서**

1. 현재 시각을 `generated_at`으로 저장한다.
2. `llm_client.generate_report_text(core_findings, config)`를 호출하여 리포트 텍스트를 얻는다.
3. (선택) 리포트 상단 첫 줄/Regime 부분을 추출해 `summary_header` 같은 필드를 만들 수도 있다.
4. `AnalystReport` 객체를 생성:

   * `generated_at = generated_at`
   * `config = config` (또는 요약본)
   * `core_findings = core_findings`
   * `formatted_text = LLM에서 받은 리포트 텍스트`
   * (옵션) `summary_header` 설정
5. `AnalystReport`를 반환한다.

---

# 6. 오케스트레이션 (`orchestrator.py`, `run_once.py`)

### 6-1. `run_analyst_cycle_once(config: AnalystAgentConfig) -> AnalystReport`

**의사코드 / 순서**

1. 로그: “Analyst cycle 시작, config 요약”을 기록.
2. 크롤러 실행 여부 확인:

   * `if config.run_crawler_before_analysis`:

     * `crawler_result = run_crawler_command(timeout_seconds=정책값)`
     * `crawler_result.success`가 False이면:

       * 로그에 경고 기록(“크롤러 실패, 기존 DB 기준으로 분석 진행”).
3. DB 조회:

   * `query_result = run_query_db_all(config.hours_window, config.news_limit, timeout_seconds=정책값)`
   * `query_result.success`가 False이면:

     * 에러 내용을 로그에 기록.
     * (정책) 여기서 예외를 발생시키거나, “이번 사이클 리포트 생성 불가” 리포트를 생성할지 결정.
4. 성공한 경우:

   * `raw_output = query_result.stdout`
5. 스냅샷 구성:

   * `snapshot = build_db_snapshot(raw_output, config.hours_window)`
6. 핵심 분석:

   * `core_findings = build_core_findings(snapshot, config)`
7. LLM 클라이언트 준비:

   * `llm_client = LlmClient(config.llm)` (생성 시 config 주입)
8. 리포트 생성:

   * `report = generate_analyst_report(core_findings, config, llm_client)`
9. 리포트 저장:

   * `save_path = save_analyst_report(report)`
10. 로깅:

    * `log_analyst_cycle(report, {"crawler": crawler_result, "query_db_all": query_result})`
11. `AnalystReport`를 반환한다.

---

### 6-2. `save_analyst_report(report: AnalystReport) -> str`

**의사코드 / 순서**

1. 출력 디렉토리 `config.report_output_dir` 존재 여부 확인:

   * 없으면 디렉토리를 생성.
2. 파일명 생성:

   * `YYYYMMDD_HHMM` 형식의 타임스탬프
   * 현재 Regime 라벨(있으면)을 붙인다:

     * 예: `analyst_report_20251203_1336_sideways.md`
3. 파일 전체 경로를 구성:

   * `os.path.join(report_output_dir, filename)`
4. 파일을 열어 `report.formatted_text` 전체를 기록한다.
5. (옵션) 동일 이름 or 별도 확장자로 JSON 직렬화 결과도 기록:

   * `analyst_report_...json` 파일에 `core_findings` + 메타데이터 저장.
6. 파일을 닫고, 최종 경로 문자열을 반환한다.

---

### 6-3. `log_analyst_cycle(report: AnalystReport, cli_results: dict[str, CliCommandResult]) -> None`

**의사코드 / 순서**

1. 로그 파일 위치를 `config.log_dir` 기준으로 설정:

   * 예: `logs/analyst_agent.log`
2. 한 사이클 요약 로그 레코드를 만든다:

   * 타임스탬프, 분석 시간창, 뉴스 개수(`report.core_findings.article_stats.total_articles`),
     현재 Regime, Volatility level, 상위 추천 종목 코드 목록 등.
3. 각 CLI 호출 결과(crawler, query_db_all)에 대해:

   * 성공 여부, exit_code, 실행 시간( finished_at - started_at )을 로그에 기록.
4. 로그 파일에 레코드를 append 한다.
5. 별도 에러 발생 시 (파일 쓰기 실패 등), stdout에 경고 출력.

---

### 6-4. `main()` (`run_once.py`)

**의사코드 / 순서**

1. `load_analyst_agent_config()` 호출 → `config` 획득.
2. `run_analyst_cycle_once(config)`를 호출하여 `report` 획득.
3. `report.formatted_text`를 표준 출력으로 출력:

   * 터미널에서 바로 리포트 확인 가능하게.
4. (선택) 종료 코드:

   * 성공적으로 리포트를 생성했다면 exit code 0.
   * 예외/에러가 있었다면 적절한 non-zero exit code로 종료.

---

이렇게 하면:

* **1단계**에서 만든 프로세스 흐름이,
* **2단계**에서 정의한 함수/데이터 타입 시그니처에 맞춰,
* **3단계**에서 “각 함수 안에서 어떤 일을 하는지”까지 모두 설계가 마무리됩니다.

이제 이 설계를 바탕으로 실제 Python 코드(파일 구조 + class/def + pass + 주석)를 만들면,
에러 없이 차근차근 구현/테스트해 나갈 수 있을 거예요.
