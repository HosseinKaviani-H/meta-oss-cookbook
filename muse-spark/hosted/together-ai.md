# Together AI

Together AI serves Muse Spark Open on serverless and on-demand dedicated endpoints through an OpenAI-compatible API.

## Links

- [Muse Spark Open on Together AI](https://www.together.ai/models/muse-spark-open) has pricing and endpoint options.

## Model string

```text
meta-models/muse-spark-open
```

## Get a key

Create an [API key](https://api.together.ai/settings/api-keys) and export it:

```bash
export TOGETHER_API_KEY="YOUR_API_KEY"
```

## Send requests

The base URL is `https://api.together.xyz/v1`. Any OpenAI-compatible client works.

```bash
curl https://api.together.xyz/v1/chat/completions \
  -H "Authorization: Bearer $TOGETHER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-models/muse-spark-open",
    "messages": [{"role": "user", "content": "Explain mixture-of-experts models in two sentences."}]
  }'
```
