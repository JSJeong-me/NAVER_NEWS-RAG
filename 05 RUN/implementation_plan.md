# News Regime Code Completion and Testing Plan

## Overview

The `news_regime` package is a Naver Finance news crawler with LLM-based analysis/recommendation system. The codebase contains well-structured pseudocode with domain models, configuration, and LLM integration already implemented. **28 methods/classes** contain `NotImplementedError` stubs that need implementation.

## Architecture Summary

The system follows this data flow:
1. **Crawl**: Fetch Naver Finance main news → Parse article metadata & details
2. **Map**: Match articles to stock symbols using dictionary/matcher
3. **Analyze**: Extract sentiment, risk flags, and impact scores
4. **Aggregate**: Roll up article-level stats by symbol and globally
5. **Recommend**: Generate buy recommendations based on aggregated stats
6. **Regime**: Classify market regime (risk_on, risk_off, volatile, etc.)
7. **Archive**: Export/clean old data after retention period

## Modules Status

### ✅ Fully Implemented
- `domain/models.py` - All dataclasses defined
- `config/models.py` - All configuration dataclasses
- `config/loader.py` - Basic config loader (returns defaults)
- `llm/client.py` - OpenAI client wrapper with JSON parsing
- `llm/prompts.py` - Prompt templates and schemas
- `llm/classifier.py` - Article classification using LLM
- `llm/summarizer.py` - Article summarization using LLM
- `common/exceptions.py` - Custom exceptions
- `common/logging.py` - Logging setup
- `common/time_utils.py` - Timezone utilities

### ⚠️ Require Implementation (Pseudocode Only)
- `crawler/http_client.py` - HTTP GET method
- `crawler/naver_client.py` - Fetch & parse methods (4 methods)
- `parser/mainnews_parser.py` - HTML parsing for main news list
- `parser/article_parser.py` - HTML parsing for article detail
- `preprocess/text_normalizer.py` - HTML stripping and text normalization (3 methods)
- `mapping/symbol_dictionary.py` - Load and find symbols (2 methods)
- `mapping/matcher.py` - Match articles to symbols
- `analysis/sentiment.py` - Sentiment scoring
- `analysis/risk.py` - Risk flag tagging
- `analysis/impact.py` - Impact scoring
- `analysis/aggregator.py` - Aggregate by symbol & globally (2 methods)
- `recommend/policy.py` - Buy candidate filter & scoring (2 methods)
- `recommend/engine.py` - Generate recommendations
- `regime/classifier.py` - Classify market regime
- `storage/db.py` - Database session management
- `storage/repositories/*.py` - All repository CRUD methods (~10 methods)
- `archive/*.py` - Archive planning, export, cleanup (4 files)
- `cli/hourly_job.py` - Main orchestration loop
- `cli/daily_archive_job.py` - Daily archive job

## Proposed Changes

### Component 1: HTTP & Crawling

#### [NEW] [requirements.txt](file:///home/ubuntu/news_regime/requirements.txt)
Create dependencies file with:
- `openai>=1.0.0` - LLM client (already used)
- `httpx>=0.24.0` - Modern HTTP client with retry support
- `beautifulsoup4>=4.12.0` - HTML parsing
- `lxml>=4.9.0` - Fast HTML parser backend

#### [MODIFY] [http_client.py](file:///home/ubuntu/news_regime/crawler/http_client.py)
Implement `get()` method using `httpx` with retry logic and rate limiting per `CrawlConfig`.

#### [MODIFY] [naver_client.py](file:///home/ubuntu/news_regime/crawler/naver_client.py)
Implement 4 methods:
- `fetch_mainnews_page()` - GET request to mainnews URL
- `fetch_article_page()` - GET request to article URL
- `crawl_mainnews()` - Fetch + parse list of NewsArticleMeta
- `crawl_article_detail()` - Fetch + parse full NewsArticleDetail

---

### Component 2: Parsing & Preprocessing

#### [MODIFY] [mainnews_parser.py](file:///home/ubuntu/news_regime/parser/mainnews_parser.py)
Implement `parse()` to extract list items from mainnews HTML using BeautifulSoup. Extract title, URL, publisher, category, published_at from each item.

#### [MODIFY] [article_parser.py](file:///home/ubuntu/news_regime/parser/article_parser.py)
Implement `parse()` to extract body_text, reporter, tags from article HTML.

#### [MODIFY] [text_normalizer.py](file:///home/ubuntu/news_regime/preprocess/text_normalizer.py)
Implement 3 methods:
- `strip_html()` - Remove HTML tags using BeautifulSoup
- `normalize_whitespace()` - Collapse multiple spaces/newlines
- `normalize_text()` - Combined pipeline

#### [NEW] [preprocess/time_utils.py](file:///home/ubuntu/news_regime/preprocess/time_utils.py)
Create `TimeParser` class to parse Korean datetime strings from Naver HTML.

---

### Component 3: Symbol Mapping

#### [MODIFY] [symbol_dictionary.py](file:///home/ubuntu/news_regime/mapping/symbol_dictionary.py)
Implement:
- `load()` - Load hardcoded list of major Korean stocks (Samsung, Hyundai, etc.)
- `find_by_symbol()` - Lookup by symbol code

#### [MODIFY] [matcher.py](file:///home/ubuntu/news_regime/mapping/matcher.py)
Implement `match_symbols()` using simple keyword matching against company names in article text.

---

### Component 4: Analysis

#### [MODIFY] [sentiment.py](file:///home/ubuntu/news_regime/analysis/sentiment.py)
Implement `RuleBasedSentimentAnalyzer.analyze()` with Korean keyword dictionaries (positive/negative words).

#### [MODIFY] [risk.py](file:///home/ubuntu/news_regime/analysis/risk.py)
Implement `RuleBasedRiskTagger.tag()` with keyword detection for regulatory/earnings/credit risks.

#### [MODIFY] [impact.py](file:///home/ubuntu/news_regime/analysis/impact.py)
Implement `ImpactScorer.score()` based on article length, headline prominence, publisher reputation.

#### [MODIFY] [aggregator.py](file:///home/ubuntu/news_regime/analysis/aggregator.py)
Implement:
- `aggregate_by_symbol()` - Group sentiments by symbol, compute avg/max stats
- `aggregate_global()` - Compute global sentiment ratios and risk metrics

---

### Component 5: Recommendation & Regime

#### [MODIFY] [policy.py](file:///home/ubuntu/news_regime/recommend/policy.py)
Implement:
- `is_buy_candidate()` - Filter based on sentiment thresholds and risk ratios
- `compute_composite_score()` - Weighted combination of sentiment and impact

#### [MODIFY] [engine.py](file:///home/ubuntu/news_regime/recommend/engine.py)
Implement `generate_recommendations()` to filter candidates, score, sort, and optionally call LLM for reason_summary.

#### [MODIFY] [classifier.py](file:///home/ubuntu/news_regime/regime/classifier.py)
Implement `RegimeClassifier.classify()` using rules on GlobalNewsStats thresholds, optionally calling LLM summarizer for explanation.

---

### Component 6: Storage (Simplified)

> [!NOTE]
> Full database implementation would require SQLAlchemy models and migrations. For testing purposes, we'll implement **in-memory storage** using Python dictionaries.

#### [MODIFY] [db.py](file:///home/ubuntu/news_regime/storage/db.py)
Implement `create_session()` returning a simple dict-based "session" object for in-memory storage.

#### [MODIFY] Repository files
Implement all repository methods using in-memory dictionaries:
- `news_repo.py` - Store NewsArticleMeta and NewsArticleDetail
- `mapping_repo.py` - Store SymbolMatch records
- `analysis_repo.py` - Store ArticleSentiment records
- `recommendation_repo.py` - Store RecommendationItem records
- `regime_repo.py` - Store RegimeStatus records

---

### Component 7: Archive & CLI

#### [MODIFY] Archive modules
Implement basic archive functionality:
- `planner.py` - Identify records older than retention_days
- `exporter.py` - Export to Parquet files
- `cleaner.py` - Delete old records
- `job.py` - Orchestrate daily archive job

#### [MODIFY] [hourly_job.py](file:///home/ubuntu/news_regime/cli/hourly_job.py)
Implement `run_once()` orchestration:
1. Crawl mainnews → Parse → Store
2. Crawl article details → Store
3. Map to symbols → Store mappings
4. Analyze sentiment/risk/impact → Store
5. Aggregate by symbol and globally
6. Generate recommendations → Store
7. Classify regime → Store

#### [NEW] [daily_archive_job.py](file:///home/ubuntu/news_regime/cli/daily_archive_job.py)
Create simple archive job runner.

---

## Testing Strategy

### Unit Tests by Module

Create `tests/` directory with unit tests for each module:

1. **`tests/test_crawler.py`**
   - Test `HttpClient.get()` with mocked httpx responses
   - Test `NaverNewsClient` methods with fixture HTML

2. **`tests/test_parser.py`**
   - Test `MainNewsParser` with sample mainnews HTML
   - Test `ArticleParser` with sample article HTML

3. **`tests/test_preprocess.py`**
   - Test `TextNormalizer` methods
   - Test `TimeParser` with Korean datetime strings

4. **`tests/test_mapping.py`**
   - Test `SymbolDictionary` load and lookup
   - Test `SymbolMatcher` keyword matching

5. **`tests/test_analysis.py`**
   - Test sentiment analyzer with sample texts
   - Test risk tagger with keyword scenarios
   - Test impact scorer
   - Test aggregation logic

6. **`tests/test_recommend.py`**
   - Test recommendation policy filters
   - Test recommendation engine scoring

7. **`tests/test_regime.py`**
   - Test regime classifier with various GlobalNewsStats

8. **`tests/test_storage.py`**
   - Test all repository CRUD operations

### Integration Test

Create `tests/test_integration.py`:
- Mock HTTP responses for Naver pages
- Run full `HourlyJob.run_once()` pipeline
- Verify recommendations and regime status are generated
- Assert data is stored in all repositories

### Test Execution

All tests will use pytest framework:
```bash
# Install test dependencies
pip install -e .
pip install pytest pytest-mock pytest-cov

# Run all unit tests
pytest tests/ -v

# Run with coverage
pytest tests/ --cov=news_regime --cov-report=term-missing

# Run specific module test
pytest tests/test_crawler.py -v

# Run integration test
pytest tests/test_integration.py -v
```

## Verification Plan

### Automated Tests
1. **Unit tests** - Test each module independently with mocked dependencies
2. **Integration test** - Test full pipeline with mocked HTTP responses
3. **Coverage target** - Aim for >80% code coverage
4. **Commands**:
   ```bash
   cd /home/ubuntu/news_regime
   pip install -e .
   pip install pytest pytest-mock pytest-cov
   pytest tests/ -v --cov=news_regime
   ```

### Manual Verification
Since this requires actual Naver Finance HTML pages, I will:
1. Create HTML fixtures from real Naver pages (saved locally)
2. Test parsers against these fixtures
3. Demonstrate the full pipeline working with fixture data

## Summary

This implementation will:
- ✅ Complete all 28 NotImplementedError methods
- ✅ Create comprehensive unit tests for each module
- ✅ Create integration test for full pipeline
- ✅ Use in-memory storage (no database required)
- ✅ Include proper error handling and logging
- ✅ Follow existing code patterns and type hints
