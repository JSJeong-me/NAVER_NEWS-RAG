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

