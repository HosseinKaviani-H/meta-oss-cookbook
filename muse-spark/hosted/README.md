# Hosted providers

Call Muse Spark Open through a hosted provider when you'd rather not provision GPU capacity yourself. These providers serve the open weights:

| Provider | Model string | Get a key |
| --- | --- | --- |
| [Together AI](together-ai.md) | `meta-models/muse-spark-open` | [Together AI API keys](https://api.together.ai/settings/api-keys) |
| [Fireworks AI](fireworks-ai.md) | `accounts/fireworks/models/muse-spark-open` | [API keys page](https://app.fireworks.ai/settings/users/api-keys) |
| [Baseten](baseten.md) | `meta-models/muse-spark-open` | [Baseten app](https://app.baseten.co), Settings → API Keys |
| [Groq](groq.md) | `meta-models/muse-spark-open` | [GroqCloud console](https://console.groq.com/keys), once your organization has access |
| [Nebius Token Factory](nebius.md) | `meta-models/muse-spark-open` | [Token Factory API keys](https://tokenfactory.nebius.com/project/api-keys) |

You can also call Muse Spark Open through [Modal](#modal) and [OpenRouter](#openrouter).

## Send a request

Each block below is one provider's request:

**Together AI**

```bash
curl https://api.together.xyz/v1/chat/completions \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-models/muse-spark-open",
    "messages": [{"role": "user", "content": "Explain mixture-of-experts models in two sentences."}]
  }'
```

**Fireworks AI**

```bash
curl https://api.fireworks.ai/inference/v1/chat/completions \
  -H "Authorization: Bearer $FIREWORKS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "accounts/fireworks/models/muse-spark-open",
    "messages": [{"role": "user", "content": "Explain mixture-of-experts models in two sentences."}],
    "max_tokens": 4096
  }'
```

**Baseten**

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["BASETEN_API_KEY"],
    base_url="https://inference.baseten.co/v1",
)

response = client.chat.completions.create(
    model="meta-models/muse-spark-open",
    messages=[{"role": "user", "content": "Explain what a stack trace is in two sentences."}],
)
print(response.choices[0].message.content)
```

**Groq**

```bash
# GroqCloud access requires approval for your organization; see Groq below.
curl https://api.groq.com/openai/v1/chat/completions \
  -H "Authorization: Bearer $GROQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-models/muse-spark-open",
    "reasoning_effort": "medium",
    "messages": [{"role": "user", "content": "What is the capital of France?"}]
  }'
```

**Nebius Token Factory**

```bash
curl https://api.tokenfactory.us-north1.nebius.com/v1/chat/completions \
  -H "Authorization: Bearer $NEBIUS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-models/muse-spark-open",
    "messages": [{"role": "user", "content": "Say hello in one sentence."}],
    "max_tokens": 16384
  }'
```

## Modal

Modal serves Muse Spark Open. See [Muse Spark Open on Modal](https://modal.com/library/meta/muse-spark-open) for setup and request examples, and the [Modal blog post on Muse Spark Open](https://modal.com/blog/meta-muse-spark-open-now-available-on-modal).

## OpenRouter

OpenRouter routes requests to upstream providers rather than hosting the model, so the provider serving your request depends on your routing configuration. See [Muse Spark Open on OpenRouter](https://openrouter.ai/meta/muse-spark-open) for the model string and setup.
