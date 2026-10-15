# Baseten

Baseten serves Muse Spark Open on Model APIs (serverless, pay per token) and Dedicated Inference (single-tenant capacity), with the full 1,048,576-token context window and text, image, video, and audio input.

## Links

- [Baseten launch blog post for Muse Spark Open](https://www.baseten.co/blog/muse-spark-open-launch/)
- [Baseten model page for Muse Spark Open](https://www.baseten.co/library/muse-spark-open)
- [Baseten docs](https://docs.baseten.co)

## Model string

```text
meta-models/muse-spark-open
```

## Get a key

1. Create an account in the [Baseten app](https://app.baseten.co).
2. Go to Settings → API Keys and create a key.

## Send requests

The endpoint is OpenAI-compatible. Point any OpenAI client at Baseten:

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

Image, video, and audio inputs use the standard OpenAI content-part format (`image_url`, `video_url`, `audio_url`). Send media larger than ~74 MB by URL rather than inline base64; video inputs by URL are supported up to 200 MB.

## More

- Pricing: see the [Baseten model page for Muse Spark Open](https://www.baseten.co/library/muse-spark-open).
- The video encoder samples at most 96 frames per clip: a 720p clip costs about 144 tokens per second, capped at ~6.9K tokens from about 48 seconds on, so longer clips (up to 300 s) cost no more than that.
