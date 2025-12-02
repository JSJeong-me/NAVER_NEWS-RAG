# News Regime - Quick Start Guide

## Installation

```bash
cd /home/ubuntu/news_regime
pip install -e .
```

## Running Tests

### All Tests
```bash
python -m pytest tests/ -v
```

### Integration Test Only
```bash
python -m pytest tests/test_integration.py -v -s
```

### Specific Module Tests
```bash
python -m pytest tests/test_analysis.py -v
python -m pytest tests/test_mapping.py -v
python -m pytest tests/test_storage.py -v
```

### With Coverage Report
```bash
python -m pytest tests/ --cov=news_regime --cov-report=term-missing
```

## Expected Results

✅ **16/16 tests passing** 

Sample output from integration test:
```
✓ Processed 2 articles
✓ Found 3 symbol matches  
✓ Analyzed 2 sentiments
✓ Generated 2 recommendations
✓ Regime: risk_on (confidence=0.80)
✓ Integration test passed!
```

## Package Structure

All modules are in `/home/ubuntu/news_regime/` which is the package root:
- `analysis/` - Sentiment, risk, impact analysis
- `crawler/` - HTTP client & Naver crawler
- `parser/` - HTML parsers
- `mapping/` - Symbol dictionary & matcher
- `recommend/` - Recommendation engine
- `regime/` - Market regime classifier
- `storage/` - In-memory repositories
- `archive/` - Data retention
- `cli/` - Hourly job orchestration
- `tests/` - Test suite

## More Information

See `walkthrough.md` for complete documentation.
