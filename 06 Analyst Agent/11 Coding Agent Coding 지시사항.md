coding agent 에서 지금 작성한 python code 를 완성할 수 있도록 '요구사항', 'code 의 목적' 및 '단계별 test' 등의 'code에 필요한 지시사항'을 영문으로 정리하여 작성해 주세요.



Here’s a clean spec you can hand to a coding agent to finish the skeleton you just created.

---

## 1. Project Context & Overall Goal

**Context**

* The project lives under: `/home/ubuntu/news_regime/`
* Existing CLI tools:

  * `./run_crawler.sh` – crawls Naver Finance main news (`https://finance.naver.com/news/mainnews.naver`).
  * `./query_db.sh all --hours H --limit N` – prints a text report of:

    * News Articles
    * Article Analysis (sentiment/impact/risk flags)
    * Symbol Mappings
    * Recommendations
    * Regime History
* A SQLite DB already exists and is managed by these scripts.

**Goal**

Implement the `analyst_agent` package so that:

1. Every 30 minutes (or on-demand via CLI), it can:

   * Optionally run the crawler,
   * Run `query_db.sh all` with a specified lookback window,
   * Parse the text output into structured Python objects,
   * Analyze market **regime**, **volatility**, **symbol focus**, and **recommendations**,
   * Call an **OpenAI Agents SDK–based LLM** to generate a human-readable **Analyst Report**,
   * Save the report to disk and log the cycle.

2. The resulting `AnalystReport` object will later be consumed by a separate **Trader Agent** (multi-agent architecture), so its structure must remain stable.

---

## 2. Functional Requirements

### 2.1 Core functional behavior

The completed Analyst Agent must:

1. **Run an analysis cycle** (`run_analyst_cycle_once`):

   * Load configuration (`AnalystAgentConfig`).
   * Optional: run `run_crawler.sh` before analysis, based on config.
   * Run `query_db.sh all --hours H --limit N`.
   * Parse the CLI output into a `DbSnapshot`.
   * Compute `AnalystCoreFindings` (regime, volatility, key symbols, key news drivers, etc.).
   * Use `LlmClient` (OpenAI Agents SDK) to generate a textual report.
   * Wrap results into an `AnalystReport`.
   * Save the report to disk.
   * Log summary info.

2. **Interpret regime and volatility**:

   * Detect when **regime changes** occur (e.g., `risk_on → sideways`).
   * Assess **volatility level** (`NORMAL`, `WATCH`, `CAUTION`, `SPIKE`) based on:

     * average impact,
     * risk flag ratios,
     * regime shifts.

3. **Identify key opportunities & risks**:

   * Rank and select **top recommendations** (e.g., up to 5).
   * Identify **focus symbols** (symbols with many articles and/or strong scores).
   * Extract **key news drivers** that move:

     * the market,
     * sectors,
     * or specific symbols.

4. **Generate LLM-based report**:

   * Use OpenAI Agents SDK to:

     * Accept `AnalystCoreFindings` as structured input,
     * Produce a structured text/markdown report that clearly highlights:

       * current regime,
       * regime changes,
       * volatility level,
       * key recommendations & symbols,
       * key market-moving news,
       * what to watch in the next session.

5. **Be robust to partial data**:

   * If some sections are missing in the CLI output (e.g., no Regime History), still produce a reasonable report.
   * If the crawler fails, continue using existing DB data.
   * If the LLM call fails, produce a minimal report with numeric/statistical info only.

---

## 3. Non-Functional Requirements

1. **Environment**

   * OS: Ubuntu 20.04 / 22.04 (Jetson AGX Orin).
   * Python: 3.1x (3.11+).
   * The implementation should not assume a virtualenv; but should be compatible with standard venv/conda.

2. **Performance & Overhead**

   * The cycle runs at most every 30 minutes.
   * Parsing and analysis must be lightweight (pure Python, no heavy libraries).
   * LLM should be called **once per cycle** (for the report only).

3. **Reliability**

   * Use defensive parsing: tolerate missing sections and small format changes.
   * Avoid crashes due to:

     * zero articles,
     * missing regime history,
     * parsing errors.
   * Use `NotImplementedError` only where truly unimplemented; after completion, tests should run without throwing these.

4. **Maintainability**

   * Respect existing **public interfaces** (dataclasses, Enums, and function signatures) in the skeleton.
   * Add helper functions if needed, but avoid changing existing type signatures unless strictly necessary.

---

## 4. Code Purpose by Module

All modules already exist as skeletons with dataclasses and signatures. Implement the TODOs and `NotImplementedError`s.

### 4.1 `config.py`

**Purpose**

* Provide configuration objects and a simple loader.

**Tasks**

* Implement `load_analyst_agent_config()`:

  * Load from:

    * Environment variables, and/or
    * Optional config file (`./config/analyst_agent.yaml` if available).
  * Apply defaults as defined in `AnalystAgentConfig`.
  * No heavy dependency (e.g., optional `yaml` import with graceful fallback).

### 4.2 `models.py`

**Purpose**

* Domain-level data structures for:

  * News articles
  * Aggregated stats
  * Symbol stats
  * Recommendations
  * Regime records
  * DbSnapshot

**Tasks**

* Already mostly done. Only needs small helper functions if you choose (e.g., for JSON serialization), but not required.

### 4.3 `report_schema.py`

**Purpose**

* Higher-level objects that represent:

  * Regime changes
  * Volatility assessment
  * Focus symbols
  * Key news drivers
  * Core findings
  * Final reports

**Tasks**

* Data structures are already defined.
* Keep them stable: they are the contract toward the future Trader Agent.

### 4.4 `cli_tools.py`

**Purpose**

* Provide safe, testable wrappers around:

  * `./run_crawler.sh`
  * `./query_db.sh all`

**Tasks**

* Implement:

  * `run_crawler_command(timeout_seconds)`
  * `run_query_db_all(hours, limit, timeout_seconds)`

**Implementation details**

* Compute project root from current file (e.g., `BASE_DIR = Path(__file__).resolve().parents[2]`).
* Use `subprocess.run` with:

  * `cwd=BASE_DIR`
  * `check=False`
  * `capture_output=True`
  * `text=True`
* Fill `CliCommandResult` with:

  * `command`
  * `stdout`, `stderr`
  * `exit_code`
  * `started_at`, `finished_at`
  * `success = (exit_code == 0)`

### 4.5 `parse_query_output.py`

**Purpose**

* Convert `query_db.sh all` raw text into a structured `DbSnapshot`.

**Tasks**

Implement:

* `split_query_all_output(raw_output) -> Dict[str, str]`

  * Detect headings like:

    * `"News Articles ("`
    * `"Article Analysis ("`
    * `"Symbol Mappings ("`
    * `"Recent Recommendations ("`
    * `"Regime History ("`
  * Group subsequent lines under keys:

    * `"news_articles"`
    * `"article_analysis"`
    * `"symbol_mappings"`
    * `"recommendations"`
    * `"regime_history"`

* `parse_news_section(section_text) -> List[NewsArticle]`

  * Parse blocks like:

    ```text
    1. Title...
       Publisher: ...
       Category: MainNews
       Crawled: 2025-12-03 13:32:02.435538+09:00
       URL: ...
       Summary: ...
    ```
  * Extract fields and convert `Crawled:` to `datetime`.

* `parse_article_analysis_section(section_text) -> ArticleStats`

  * Parse lines like:

    * `Total Articles Analyzed: 40`
    * `Average Sentiment Score: 0.275`
    * etc.
  * Return `ArticleStats`. If parsing fails, return zeros and log a warning.

* `parse_symbol_mappings_section(section_text) -> List[SymbolStats]`

  * Lines like:

    * `005930: 5 articles (avg score: 0.580)`
  * Extract `symbol`, `article_count`, `avg_score`.

* `parse_recommendations_section(section_text) -> List[Recommendation]`

  * Block format, e.g.:

    ```text
    1. 035420 - ACTIVE
       Created: 2025-12-03 13:32:02.435538+09:00
       Sentiment: +1.000 | Impact: 0.200 | Composite: 0.680
       Risks: {'regulatory_risk_ratio': 0.0, ...}
       Reason: 긍정적 뉴스 1건, 평균 심리지수 1.00
    ```
  * Parse each field, including the dict string in `Risks:` (safe `ast.literal_eval`).

* `parse_regime_history_section(section_text) -> List[RegimeRecord]`

  * E.g.:

    ```text
    1. 2025-12-03 13:36:34.918289+09:00 - sideways (confidence: 60.00%)
       중립적 시장 심리
    ```
  * Extract timestamp, label, confidence, description.

* `build_db_snapshot(raw_output, hours_window) -> DbSnapshot`

  * Orchestrate all parsers and return a `DbSnapshot`.

**Important**: Be defensive:

* If a section is missing, return empty lists/defaults instead of throwing.

### 4.6 `analysis_logic.py`

**Purpose**

* From `DbSnapshot` and config, produce high-level `AnalystCoreFindings`.

**Tasks**

Implement:

* `analyze_regime_change(regime_history, window_size) -> RegimeChange`

  * Sort by timestamp.
  * Determine `current`, `previous`.
  * If no previous, `has_changed=False`.
  * Else, compare labels; if different, `has_changed=True` and `change_direction="prev_to_curr"`.

* `assess_volatility(article_stats, regime_history, config) -> VolatilityAssessment`

  * Use:

    * `article_stats.avg_impact`
    * risk flag ratios
    * recent regime shift
  * Map to `VolatilityLevel` using `config.volatility_thresholds`.
  * Explain decisions in `reason` and `key_drivers`.

* `select_top_recommendations(recommendations, max_count) -> List[Recommendation]`

  * Filter ACTIVE.
  * Sort by `composite` descending.
  * Slice to `max_count`.

* `identify_focus_symbols(symbol_stats, recommendations, max_count) -> List[FocusSymbol]`

  * Sort `symbol_stats` by `article_count` (and maybe `avg_score`).
  * Mark `is_recommended` if symbol appears in recommendations.
  * Add `notes` summarizing article_count, avg_sentiment, recommendation status.

* `extract_key_news_drivers(news, article_stats, regime_history, max_count) -> List[KeyNewsDriver]`

  * Heuristic selection:

    * Keywords such as “거래세 인상”, “코스피”, “관세”, “정보유출”, “AI 특수”, etc.
  * Decide `impact_scope` (`market`, `sector`, `stock`).
  * Decide `impact_direction` (`positive`, `negative`, `mixed`).

* `build_core_findings(snapshot, config) -> AnalystCoreFindings`

  * Combine all of the above into one `AnalystCoreFindings`.

### 4.7 `llm_client.py`

**Purpose**

* Provide a clean abstraction over OpenAI Agents SDK.

**Tasks**

* Implement `LlmClient.generate_report_text(core_findings, extra_context=None) -> str`:

  * Convert `core_findings` to a compact dictionary.
  * Build:

    * system message: define role as a market news Analyst.
    * user message: include structured JSON and instructions:

      * sections: Regime, Volatility, Recommendations, Focus Symbols, Key Drivers, Summary.
    * instructions:

      * short, actionable, clearly highlight regime changes and volatility spikes.
  * Call the OpenAI Agents SDK and return the text body.
  * Handle exceptions and timeouts gracefully.

### 4.8 `report_generator.py`

**Purpose**

* Turn `core_findings` + LLM text into an `AnalystReport`.

**Tasks**

* Implement `generate_analyst_report(core_findings, config, llm_client) -> AnalystReport`:

  * `generated_at = now()`.
  * `formatted_text = llm_client.generate_report_text(core_findings)`.
  * Optionally derive `summary_header` from the first section.
  * Create and return `AnalystReport`.

### 4.9 `orchestrator.py`

**Purpose**

* High-level orchestration of a single analysis cycle.

**Tasks**

* Implement `run_analyst_cycle_once(config) -> AnalystReport`:

  * Optionally call `run_crawler_command`.
  * Call `run_query_db_all(config.hours_window, config.news_limit)`.
  * If query fails: raise or fallback (see requirements).
  * Build `DbSnapshot` via `build_db_snapshot`.
  * Build `AnalystCoreFindings` via `build_core_findings`.
  * Instantiate `LlmClient(config.llm)`.
  * Call `generate_analyst_report`.
  * Call `save_analyst_report`.
  * Call `log_analyst_cycle`.
  * Return `AnalystReport`.

* Implement `save_analyst_report(report, output_dir) -> Path`:

  * Create directory if missing.
  * File name like: `analyst_report_YYYYMMDD_HHMM_<regime>.md`.
  * Write `report.formatted_text`.
  * Optionally save a JSON version of `core_findings`.

* Implement `log_analyst_cycle(report, cli_results, log_dir)`:

  * Append a log line to e.g. `logs/analyst_agent.log` with:

    * timestamp,
    * current regime label & confidence,
    * volatility level,
    * top recommendation symbols,
    * CLI success flags and durations.

### 4.10 `run_once.py`

**Purpose**

* Module entrypoint to run a single cycle.

**Tasks**

* `main(argv=None)`:

  * Call `load_analyst_agent_config()`.
  * Call `run_analyst_cycle_once(config)`.
  * Print `report.formatted_text` to stdout.
  * Save report via `save_analyst_report`.
  * Return exit code 0 (or non-zero on fatal failure).
* Keep CLI parsing minimal or optional.

---

## 5. Step-by-Step Development & Test Plan

The coding agent should implement and test in small, verifiable steps.

### Step 0 – Environment Smoke Test

* Ensure package import works (after copying under `/home/ubuntu/news_regime`):

```bash
cd /home/ubuntu/news_regime
python -c "import analyst_agent; print('ok:', analyst_agent.__name__)"
```

Expected: `ok: analyst_agent`

---

### Step 1 – Config & Basic Skeleton

**Implement**

* `load_analyst_agent_config()` returning defaults.

**Test**

```bash
python -c "from analyst_agent.config import load_analyst_agent_config; print(load_analyst_agent_config())"
```

Expected: prints a config object with default fields.

---

### Step 2 – CLI Tools

**Implement**

* `run_crawler_command()`
* `run_query_db_all()`

**Test**

From `/home/ubuntu/news_regime`:

```bash
python - << 'EOF'
from analyst_agent.cli_tools import run_query_db_all
res = run_query_db_all(hours=24, limit=10)
print("success:", res.success, "exit_code:", res.exit_code)
print("stdout_head:", res.stdout[:500])
EOF
```

Expected:

* `success` is True.
* `stdout_head` begins with the formatted “News Articles …” section.

---

### Step 3 – Parsing Logic

**Implement**

* `split_query_all_output`
* `parse_*_section` functions
* `build_db_snapshot`

**Test**

Use a captured `query_db.sh all --hours 24 --limit 10` output (e.g. from the user’s docstring) as fixture:

```bash
python - << 'EOF'
import pathlib
from analyst_agent.parse_query_output import build_db_snapshot

text = pathlib.Path("sample_query_all_output.txt").read_text(encoding="utf-8")
snapshot = build_db_snapshot(text, hours_window=24)
print("news:", len(snapshot.news))
print("article_stats:", snapshot.article_stats)
print("symbols:", [s.symbol for s in snapshot.symbol_stats])
print("recs:", [(r.symbol, r.status) for r in snapshot.recommendations])
print("regimes:", [(r.label, r.confidence) for r in snapshot.regime_history])
EOF
```

Expected:

* Correct counts and some meaningful values (non-empty lists when data exists).

---

### Step 4 – Analysis Logic

**Implement**

* `analyze_regime_change`
* `assess_volatility`
* `select_top_recommendations`
* `identify_focus_symbols`
* `extract_key_news_drivers`
* `build_core_findings`

**Test**

Use a real snapshot or a handcrafted one:

```bash
python - << 'EOF'
from analyst_agent.config import load_analyst_agent_config
from analyst_agent.parse_query_output import build_db_snapshot
from analyst_agent.analysis_logic import build_core_findings

import pathlib
cfg = load_analyst_agent_config()
text = pathlib.Path("sample_query_all_output.txt").read_text(encoding="utf-8")
snapshot = build_db_snapshot(text, hours_window=24)

core = build_core_findings(snapshot, cfg)
print("regime:", core.current_regime)
print("volatility:", core.volatility.level, core.volatility.reason)
print("top_recs:", [r.symbol for r in core.top_recommendations])
print("focus:", [f.symbol for f in core.focus_symbols])
print("drivers:", [d.title for d in core.key_news_drivers])
EOF
```

Expected:

* No exceptions.
* Reasonable output (e.g., regime not None if history exists).

---

### Step 5 – LLM Client (Stub & Real)

**Phase 5a – Stub implementation**

Initially, implement `LlmClient.generate_report_text()` as a deterministic stub (no real API call):

* Create a simple text report summarizing:

  * Current regime & volatility level,
  * Top recommendations,
  * Focus symbols,
  * Key drivers.

This allows end-to-end testing without OpenAI credentials.

**Test**

```bash
python - << 'EOF'
from analyst_agent.config import load_analyst_agent_config
from analyst_agent.llm_client import LlmClient
from analyst_agent.report_generator import generate_analyst_report
from analyst_agent.parse_query_output import build_db_snapshot
from analyst_agent.analysis_logic import build_core_findings

import pathlib

cfg = load_analyst_agent_config()
text = pathlib.Path("sample_query_all_output.txt").read_text(encoding="utf-8")
snapshot = build_db_snapshot(text, hours_window=24)
core = build_core_findings(snapshot, cfg)

client = LlmClient(cfg.llm)
report = generate_analyst_report(core, cfg, client)
print(report.formatted_text)
EOF
```

Expected: A text report with all sections present (no crash).

**Phase 5b – Integrate OpenAI Agents SDK**

Once stub works:

* Replace stub logic with real API calls using the configured `model`.
* Keep interface identical.

---

### Step 6 – Orchestrator & Entry Point

**Implement**

* `run_analyst_cycle_once`
* `save_analyst_report`
* `log_analyst_cycle`
* Wire everything in `run_once.main`.

**Test: E2E**

From `/home/ubuntu/news_regime`:

```bash
python -m analyst_agent.run_once
```

Expected:

* Runs crawler if enabled.
* Runs `query_db.sh all`.
* Produces a report printed to stdout.
* Saves a report file under `./reports/`.
* Appends to a log file under `./logs/`.

---

## 6. Coding Style & Constraints for the Coding Agent

* Use **type hints** consistently.
* Do **not** remove or rename existing dataclasses, Enums, or function signatures.
* You **may add**:

  * Private helper functions (`_helper_name`),
  * Additional small modules if needed.
* Handle errors:

  * Use Python exceptions where necessary,
  * Prefer graceful degradation for partial data,
  * Only abort the whole cycle on truly critical failures (e.g., query_db cannot be run at all).
* Logging:

  * Use standard `logging` module (INFO/ERROR/WARNING).
  * Don’t spam logs; one cycle summary is enough.

---

If you give this document to a coding agent, it should have all the context and step-by-step guidance needed to safely turn the current skeleton into a fully working Analyst Agent, with minimal surprises and good testability.
