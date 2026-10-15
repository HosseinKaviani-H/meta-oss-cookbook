# Nebius Token Factory

Nebius Token Factory serves Muse Spark Open on a serverless endpoint with an OpenAI-compatible API. You pay per token. To send a request, you need an API key and the model ID below.

## Links

- [Muse Spark Open on Token Factory](https://tokenfactory.nebius.com/?modals=endpoint-details&model-id=meta-models/muse-spark-open) has current endpoint details, pricing, and availability.
- [Token Factory Playground](https://tokenfactory.nebius.com/playground?models=meta-models/muse-spark-open) lets you try the model in the browser.

## Model string

The model ID is case-sensitive:

```text
meta-models/muse-spark-open
```

Set your client's base URL to:

```text
https://api.tokenfactory.us-north1.nebius.com/v1/
```

## Get a key

1. [Create a Token Factory account](https://tokenfactory.nebius.com/) or sign in.
2. Complete [billing setup](https://docs.tokenfactory.nebius.com/other-capabilities/billing-new). New accounts need a bank card.
3. Select your project and open [API keys](https://tokenfactory.nebius.com/project/api-keys).
4. Create a key and copy it. You can't view it again later.
5. Export the key:

```bash
export NEBIUS_API_KEY="YOUR_API_KEY"
```

Keep the key out of source code and browser applications. Check that the key can list models:

```bash
curl --fail-with-body --silent --show-error \
  https://api.tokenfactory.us-north1.nebius.com/v1/models \
  -H "Authorization: Bearer $NEBIUS_API_KEY"
```

Look for `meta-models/muse-spark-open` in the response. If it's missing, check model availability for the project that owns your key.

## Send requests

```bash
curl --fail-with-body --silent --show-error \
  https://api.tokenfactory.us-north1.nebius.com/v1/chat/completions \
  -H "Authorization: Bearer $NEBIUS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-models/muse-spark-open",
    "messages": [
      {"role": "user", "content": "Say hello in one sentence."}
    ],
    "max_tokens": 16384
  }'
```

The answer is in `choices[0].message.content`.

With the OpenAI Python SDK:

```bash
pip install openai
```

```python
import os

from openai import OpenAI

client = OpenAI(
    api_key=os.environ["NEBIUS_API_KEY"],
    base_url="https://api.tokenfactory.us-north1.nebius.com/v1/",
)

response = client.chat.completions.create(
    model="meta-models/muse-spark-open",
    messages=[{"role": "user", "content": "Say hello in one sentence."}],
    max_tokens=16384,
)

print(response.choices[0].message.content)
```

**Tool calling.** Pass `tools` in the OpenAI function-calling format. This request asks for a `get_weather` call with a `city` argument:

```python
# Uses the client from the first Python example in this section
response = client.chat.completions.create(
    model="meta-models/muse-spark-open",
    messages=[{"role": "user", "content": "What is the weather in Paris?"}],
    tools=[{
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get the current weather for a city.",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"],
                "additionalProperties": False,
            },
        },
    }],
    tool_choice={"type": "function", "function": {"name": "get_weather"}},
    max_tokens=16384,
)

for call in response.choices[0].message.tool_calls or []:
    print(call.id, call.function.name, call.function.arguments)
```

The request doesn't fetch the weather itself. Your application runs the function and sends the result back with the returned call ID. See the [Chat Completions reference](https://docs.tokenfactory.nebius.com/api-reference/inference/create-chat-completion) for the `tools` and tool-message fields.

## More

- [Token Factory quickstart](https://docs.tokenfactory.nebius.com/quickstart) covers account setup and your first request.
- [API documentation](https://docs.tokenfactory.nebius.com/api-reference/examples/overview) lists endpoints, request parameters, and response formats.
- [Token Factory Cookbook](https://github.com/nebius/token-factory-cookbook) has reusable code examples and sample applications.
