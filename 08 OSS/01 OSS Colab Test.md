이전에 작성한 'news regime' agent 구현에 사용한 OpenAI GPT model 을 OSS model 교체하여 구현하려 합니다. 우선 OSS 120B model 을 test 하는 예제 program 을 colab 에서 실행할 수 있도록 작성해 주세요.



Colab Test

아래는 **Colab에서 `openai/gpt-oss-120b`를 “가능하면 120B로”, 불가능하면 자동으로 `gpt-oss-20b`로 폴백**해서 **프롬프트/출력 품질 테스트**를 할 수 있는 **실행 예제(셀 단위)**입니다.

> 현실 체크: `gpt-oss-120b`는 **단일 80GB GPU(H100/MI300X급)**를 전제로 설계된 모델이라, **대부분의 Colab GPU(T4/L4/A100 40GB)** 에서는 **로딩 중 OOM**이 날 가능성이 큽니다. ([Hugging Face][1])
> 그래서 아래 예제는 **GPU 메모리를 보고 120B가 무리면 자동으로 20B로 폴백**합니다. (20B로도 “harmony 포맷/워크플로우” 검증은 충분히 가능합니다.) ([Hugging Face][1])

---

## Colab 예제 코드 (셀 단위로 그대로 붙여넣기)

### (Cell 1) 환경/라이브러리 설치

```python
!pip -q install -U "torch" "transformers" "accelerate" "kernels" "huggingface_hub"
```

`gpt-oss` 모델 카드는 Transformers 기반 사용을 안내합니다. ([Hugging Face][1])

---

### (Cell 2) GPU 확인 + 모델 선택(120B 우선, 불가하면 20B 폴백)

```python
import torch

def get_gpu_mem_gb() -> float:
    if not torch.cuda.is_available():
        return 0.0
    props = torch.cuda.get_device_properties(0)
    return props.total_memory / (1024**3)

gpu_mem = get_gpu_mem_gb()
print(f"CUDA available: {torch.cuda.is_available()} | GPU mem: {gpu_mem:.1f} GB")

# 기본은 120B를 시도하되, Colab GPU 메모리가 작으면 20B로 자동 폴백
MODEL_120B = "openai/gpt-oss-120b"
MODEL_20B  = "openai/gpt-oss-20b"

# 120B는 통상 80GB급 GPU가 권장(모델 카드 기준) :contentReference[oaicite:3]{index=3}
USE_120B_IF_POSSIBLE = True
MIN_MEM_FOR_120B_GB = 78  # 여유 버퍼 포함 (보수적으로)

if USE_120B_IF_POSSIBLE and gpu_mem >= MIN_MEM_FOR_120B_GB:
    model_id = MODEL_120B
else:
    model_id = MODEL_20B
    if USE_120B_IF_POSSIBLE:
        print("⚠️ Colab GPU 메모리가 120B에 부족해 보입니다. gpt-oss-20b로 폴백합니다.")

print("Selected model:", model_id)
```

---

### (Cell 3) (선택) Hugging Face 로그인

모델 다운로드가 막히거나 rate-limit 걸리면 HF 토큰이 필요할 수 있습니다.

```python
from huggingface_hub import login
import os

# Colab Secrets에 HF_TOKEN을 넣어둔 경우 자동 로그인
hf_token = os.environ.get("HF_TOKEN", "").strip()
if hf_token:
    login(token=hf_token)
    print("✅ Logged in to Hugging Face.")
else:
    print("ℹ️ HF_TOKEN이 없으면 공개 리포는 그대로 진행됩니다. (문제 시 HF_TOKEN 설정 후 재실행)")
```

---

### (Cell 4) Transformers 파이프라인 로딩 (harmony 포맷 자동 적용)

`gpt-oss`는 **harmony 응답 포맷** 기반이라, **Transformers의 chat template**을 쓰면 자동 적용됩니다. ([Hugging Face][1])

```python
from transformers import pipeline

pipe = pipeline(
    "text-generation",
    model=model_id,
    torch_dtype="auto",
    device_map="auto",
)

def chat(messages, max_new_tokens=256, temperature=0.2):
    # do_sample=False로 두면 재현성 높은 테스트 가능
    out = pipe(
        messages,
        max_new_tokens=max_new_tokens,
        temperature=temperature,
        do_sample=(temperature > 0),
    )
    gen = out[0]["generated_text"]
    # Transformers chat pipeline은 generated_text가 "메시지 리스트"로 나오는 경우가 많음
    if isinstance(gen, list) and len(gen) > 0:
        last = gen[-1]
        if isinstance(last, dict) and "content" in last:
            return last["content"]
        return str(last)
    return str(gen)

print("✅ Pipeline ready.")
```

---

### (Cell 5) 테스트 1: “뉴스 요약 + 시장 레짐 힌트” (JSON 출력)

```python
sample_headlines = [
    "코스피 장중 4010선 회복…방산·원전주 강세",
    "거래세 인상 논의…2차전지·레버리지·바이오 직격탄 우려",
    "삼성·SK 반도체 업황 개선 기대…증권가 목표가 상향",
]

messages = [
    {"role": "system", "content": "You are a market analyst. Reasoning: medium. Output strictly valid JSON."},
    {"role": "user", "content": f"""
Given these Korean market headlines, produce a compact JSON report.
Headlines:
- {sample_headlines[0]}
- {sample_headlines[1]}
- {sample_headlines[2]}

Return JSON with keys:
regime: one of ["risk_on","risk_off","sideways","high_vol"]
confidence: 0-100 integer
summary_ko: 2-3 sentences in Korean
volatility_alert: boolean
drivers: list of 3 short Korean phrases
"""},
]

print(chat(messages, max_new_tokens=300, temperature=0.0))
```

> `Reasoning: medium/high/low` 같은 “추론 노력” 지시를 시스템 프롬프트로 조절 가능하다고 모델 카드가 안내합니다. ([Hugging Face][1])

---

### (Cell 6) 테스트 2: “종목 추천 후보 스코어링” (JSON 출력)

```python
candidate_symbols = [
    {"symbol": "005930", "name": "삼성전자", "news_hint": "반도체 업황 개선 기대, 목표가 상향"},
    {"symbol": "035420", "name": "NAVER", "news_hint": "AI/플랫폼 모멘텀, 순매수 유입"},
    {"symbol": "207940", "name": "삼성바이오로직스", "news_hint": "정책/규제 변수 존재, 변동성 확대 가능"},
]

messages = [
    {"role": "system", "content": "You are an equity analyst. Reasoning: medium. Output strictly valid JSON."},
    {"role": "user", "content": f"""
Score these candidates for a short-term BUY watchlist based on the provided news hints only.
Candidates:
{candidate_symbols}

Return JSON:
items: list of objects with
- symbol, name
- sentiment_score: -1.0~+1.0
- impact_score: 0.0~1.0
- risk_flags: list from ["REG","EARN","CREDIT","MACRO","GEO","NONE"]
- composite: 0.0~1.0
- rationale_ko: 1 sentence Korean
"""},
]

print(chat(messages, max_new_tokens=450, temperature=0.0))
```

---

## 다음 단계로 “news_regime 교체 테스트”에 바로 쓰는 팁

* **동일 프롬프트(요약/분류 JSON)**를 `gpt-oss-20b`로 먼저 안정화 →
  이후 120B 실행 가능한 서버(H100 80GB 등)에서 **동일 입력으로 품질 비교**가 가장 빠릅니다. ([Hugging Face][1])
* `gpt-oss`는 **harmony 포맷**이 중요하니, Colab에서도 **Transformers chat template** 방식(위 예제처럼 messages 전달)을 유지하세요. ([Hugging Face][1])

원하시면, 다음 턴에:

* **news_regime의 `llm.client` 인터페이스를 “OpenAI SDK → 로컬/OSS(Transformers/vLLM)”로 교체**할 때,
* **동일 JSON 스키마/에러 처리/재시도/타임아웃**까지 포함한 “드롭인 교체 설계”를 바로 잡아드릴게요.

[1]: https://huggingface.co/openai/gpt-oss-120b "openai/gpt-oss-120b · Hugging Face"
