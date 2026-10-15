# Groq

GroqCloud serves Muse Spark Open through an OpenAI-compatible API. Access requires approval for your organization; to request it, [contact the Groq team](https://groq.com/muse-spark-open).

## Links

- [Muse Spark Open on GroqCloud](https://console.groq.com/docs/model/meta-models/muse-spark-open)
- [Groq blog post on Muse Spark Open](https://groq.com/blog/meta-s-muse-spark-open-release-is-now-in-limited-preview-on-groqcloud)
- [Groq reasoning guide](https://console.groq.com/docs/reasoning)
- [Groq SDKs](https://console.groq.com/docs/libraries)

## Model string

```text
meta-models/muse-spark-open
```

On GroqCloud the model has a 131,072-token context window and supports tool calling, structured outputs, and image input.

## Get a key

Once your organization has access, create an API key in the [GroqCloud console](https://console.groq.com/keys) and export it:

```bash
export GROQ_API_KEY="YOUR_API_KEY"
```

## Send requests

The base URL is `https://api.groq.com/openai/v1`. Use a Groq SDK, or any OpenAI SDK with `base_url` set to that URL.

```bash
curl https://api.groq.com/openai/v1/chat/completions \
  -H "Authorization: Bearer $GROQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-models/muse-spark-open",
    "reasoning_effort": "medium",
    "messages": [{"role": "user", "content": "What is the capital of France?"}]
  }'
```

`reasoning_effort` accepts `low`, `medium`, `high`, or `xhigh`, and defaults to `xhigh`. Lower settings respond faster. See [Reasoning strength](../inference-server/settings.md#reasoning-strength).

## More

For access questions, contact [Groq sales](mailto:sales@groq.com).
