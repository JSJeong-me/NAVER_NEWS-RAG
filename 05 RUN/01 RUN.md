Repo Overview

Root package news_regime (installable via setup.py) with top-level CLI scripts in cli/hourly_job.py, cli/daily_archive_job.py, cli/list_news.py plus helper runner run_crawler.sh.
Data/config: config/models.py defines AppConfig (db, crawl, analysis, regime, archive, llm settings); config/loader.py reads TOML/env. Domain types live in domain/models.py.
Crawling/parsing pipeline: crawler/http_client.py + crawler/naver_client.py fetch pages; parser/mainnews_parser.py and parser/article_parser.py extract articles; preprocess/text_normalizer.py & preprocess/time_utils.py clean text/times.
Analysis layer: analysis/sentiment.py, analysis/risk.py, analysis/impact.py produce scores/flags; analysis/aggregator.py aggregates per symbol/global.
Mapping/recommendation/regime: mapping/symbol_dictionary.py + mapping/matcher.py map articles to symbols; recommend/policy.py + recommend/engine.py generate recommendations; regime/classifier.py classifies market regime.
LLM helpers: llm/client.py (OpenAI), llm/summarizer.py, llm/classifier.py, llm/prompts.py.
Storage: storage/db.py manages SQLite/memory sessions; repositories for each table in storage/repositories/*.py (news, symbol mappings, analysis results, recommendations, regime history). Local DB artifact news_regime.db in root.
Archiving: archive/job.py, archive/planner.py, archive/cleaner.py, archive/exporter.py handle retention/export; archive/daily_archive_job.py orchestrates daily cleanup.
API surface: api/service.py exposes higher-level service methods; analysis folder etc form main package.
Tests in tests/ cover crawler, parser, mapping, analysis, storage, and full integration (tests/test_integration.py stitches crawl→parse→map→analyze→recommend→regime).




## (base) ubuntu@ubuntu:~/news_regime$ cd /home/ubuntu
## (base) ubuntu@ubuntu:~$ export PYTHONPATH=$PYTHONPATH:/home/ubuntu && python news_regime/cli/list_news.py 2>&1 | head -5
DEBUG: Fetched 0 articles

(base) ubuntu@ubuntu:~/news_regime$ ./query_db.sh symbols
================================================================================
Symbol Mappings (3 found)
================================================================================

Symbols by Article Count:
  005930: 2 articles (avg score: 0.300)
  207940: 1 articles (avg score: 1.000)


(base) ubuntu@ubuntu:~/news_regime$ ./query_db.sh analysis
================================================================================
Article Analysis (Last 24 hours, 20 found)
================================================================================

Statistics:
  Total Articles Analyzed: 20
  Average Sentiment Score: 0.300
  Average Impact Score: 0.130
  Regulatory Risk Flags: 0
  Earnings Risk Flags: 2
  Credit Risk Flags: 0

Top 10 by Impact Score:
  1. News ID: b52a081ece1c... | Sentiment: +0.000 | Impact: 0.200 | Risks: None
  2. News ID: 1ce60c535e22... | Sentiment: +1.000 | Impact: 0.200 | Risks: None
  3. News ID: e50999fa644e... | Sentiment: +0.000 | Impact: 0.200 | Risks: None
  4. News ID: 634caf0492b2... | Sentiment: +0.000 | Impact: 0.200 | Risks: None
  5. News ID: 80321ffe2b0b... | Sentiment: +1.000 | Impact: 0.200 | Risks: None
  6. News ID: 0755f9793f62... | Sentiment: +0.000 | Impact: 0.200 | Risks: None
  7. News ID: 01815bd663e0... | Sentiment: +1.000 | Impact: 0.200 | Risks: None
  8. News ID: cb2d9c828556... | Sentiment: +0.000 | Impact: 0.200 | Risks: None
  9. News ID: 6f1857ad291b... | Sentiment: +1.000 | Impact: 0.200 | Risks: EARN
  10. News ID: 4aa61129d4b2... | Sentiment: +0.000 | Impact: 0.200 | Risks: None
(base) ubuntu@ubuntu:~/news_regime$ ./query_db.sh symbols
================================================================================
Symbol Mappings (3 found)
================================================================================

Symbols by Article Count:
  005930: 2 articles (avg score: 0.300)
  207940: 1 articles (avg score: 1.000)
(base) ubuntu@ubuntu:~/news_regime$ ./query_db.sh recommendations
================================================================================
Recent Recommendations (1 found)
================================================================================

1. 005930 - ACTIVE
   Created: 2025-12-03 08:08:19.978968+09:00
   Sentiment: +0.500 | Impact: 0.100 | Composite: 0.490
   Risks: {'regulatory_risk_ratio': 0.0, 'earnings_risk_ratio': 0.0, 'credit_risk_ratio': 0.0}
   Reason: 긍정적 뉴스 2건, 평균 심리지수 0.50


(base) ubuntu@ubuntu:~/news_regime$ ./query_db.sh regime
================================================================================
Regime History (1 found)
================================================================================

1. 2025-12-03 08:08:19.978968+09:00 - risk_on (confidence: 80.00%)
   전반적으로 긍정적 뉴스, 낮은 리스크 비율
