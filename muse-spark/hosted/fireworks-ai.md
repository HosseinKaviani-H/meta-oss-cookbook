# Fireworks AI

Fireworks serves Muse Spark Open on serverless inference. You pay per token and send requests to an OpenAI-compatible endpoint. There is no deployment to provision. The model accepts text and images, supports tool calling, and has a context window of 1,048,576 tokens. You can set how much the model reasons on each request with `reasoning_effort`.

For dedicated capacity, guaranteed throughput, or private networking, run the same model on an on-demand deployment.

## Links

- [Muse Spark Open on Fireworks](https://fireworks.ai/models/fireworks/muse-spark-open) is the model page, with pricing and a playground.
- [Fireworks blog post on Muse Spark Open](https://fireworks.ai/blog/meta-muse-spark-open)
- [Fireworks quickstart](https://docs.fireworks.ai/getting-started/quickstart) covers account setup and your first request.
- [Post-train with Fireworks AI](../post-training/fireworks-ai.md) covers LoRA fine-tuning and reinforcement learning on Muse Spark Open.

## Model string

```text
accounts/fireworks/models/muse-spark-open
```

## Get a key

1. Create a Fireworks account at [fireworks.ai](https://fireworks.ai).
2. Create an API key on the [API keys page](https://app.fireworks.ai/settings/users/api-keys), or run `firectl api-key create`.
3. Export the key:

```bash
export FIREWORKS_API_KEY="fw_..."
```

## Send requests

The base URL is `https://api.fireworks.ai/inference/v1`. Any OpenAI-compatible client works.

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

With the OpenAI Python SDK:

```bash
pip install openai
```

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["FIREWORKS_API_KEY"],
    base_url="https://api.fireworks.ai/inference/v1",
)

response = client.chat.completions.create(
    model="accounts/fireworks/models/muse-spark-open",
    messages=[{"role": "user", "content": "Explain mixture-of-experts models in two sentences."}],
    max_tokens=4096,
)
print(response.choices[0].message.content)
```

**Reasoning effort.** Muse Spark Open always reasons. Set `reasoning_effort` to `low`, `medium`, `high`, or `xhigh`. The default is `xhigh`. Higher settings think longer and cost more output tokens. Use `xhigh` for hard reasoning, coding, and agentic tasks. Reasoning is returned in `reasoning_content`, separate from `content`. Thinking cannot be turned off, so `reasoning_effort="none"` is rejected.

```python
# Uses the client from the first Python example in this section
response = client.chat.completions.create(
    model="accounts/fireworks/models/muse-spark-open",
    messages=[{"role": "user", "content": "How many primes are there below 1000?"}],
    reasoning_effort="xhigh",
    max_tokens=32768,
)
```

**Images.** Pass image URLs or base64 data URLs in the message content.

```python
# Uses the client from the first Python example in this section
response = client.chat.completions.create(
    model="accounts/fireworks/models/muse-spark-open",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What is in this image?"},
            {"type": "image_url", "image_url": {"url": "http://images.cocodataset.org/val2017/000000039769.jpg"}},
        ],
    }],
)
```

**Tool calling.** Pass `tools` in the OpenAI function-calling format. Tool calls come back in `message.tool_calls`.

```python
# Uses the client from the first Python example in this section
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get the current weather for a city.",
        "parameters": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
        },
    },
}]

response = client.chat.completions.create(
    model="accounts/fireworks/models/muse-spark-open",
    messages=[{"role": "user", "content": "What's the weather in Menlo Park?"}],
    tools=tools,
)
print(response.choices[0].message.tool_calls)
```

Streaming (`stream=True`) and JSON mode (`response_format={"type": "json_object"}`) work the same way they do for other Fireworks chat models.

## More

- Serverless is billed per token for input, cached input, and output. Current rates are on the [model page](https://fireworks.ai/models/fireworks/muse-spark-open).
- Prompt caching is automatic. Repeated prefixes, such as long system prompts and agent histories, are billed at the cached-input rate.
- For dedicated GPUs, higher rate limits, or reserved throughput, create an [on-demand deployment](https://docs.fireworks.ai/guides/ondemand-deployments) of the same model, or [contact Fireworks](https://fireworks.ai/contact).
- Fireworks also serves an [Anthropic-compatible Messages API](https://docs.fireworks.ai/tools-sdks/anthropic-compatibility), so Muse Spark Open can run in coding agents built for that API.
