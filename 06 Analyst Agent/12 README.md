# Analyst Agent Skeleton for `news_regime`

This directory contains a skeleton implementation of the Naver Finance
news-based Analyst Agent intended to live under `/home/ubuntu/news_regime`.

- All dataclasses / enums are defined.
- All public function signatures are in place.
- Internal logic is intentionally omitted and replaced with
  `NotImplementedError` plus comments so you can fill them in step by step.

To integrate:

```bash
# On your Jetson
cp -r analyst_agent /home/ubuntu/news_regime/

cd /home/ubuntu/news_regime
python -m analyst_agent.run_once
```

You can then progressively implement each TODO and add tests around them.
