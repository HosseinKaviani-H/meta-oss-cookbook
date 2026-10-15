# Unsloth

Run Muse Spark Open locally with Unsloth Desktop and Unsloth Dynamic GGUFs. Use the desktop interface to download and chat with the model, or serve an authenticated API for your applications and coding agents. Unsloth runs on macOS, Windows, Linux, and WSL, with CPU and GPU inference powered by llama.cpp.

Muse Spark Open is Meta's 816B-parameter mixture-of-experts model with about 42B active parameters, built for multimodal reasoning, coding, and agentic workflows. It supports text, image, video, and audio input with text output. This recipe covers text chat and API serving; multimodal input depends on the inference backend and the model assets you load.

## Links

- [Muse Spark Open: run locally with Unsloth](https://unsloth.ai/docs/models/muse-spark-open)
- [Muse Spark Open Unsloth Dynamic GGUFs](https://huggingface.co/unsloth/Muse-Spark-Open-816B-A42B-GGUF)
- [Download Unsloth Desktop](https://unsloth.ai/download)
- [Unsloth API guide](https://unsloth.ai/docs/basics/api)
- [Unsloth Dynamic GGUFs](https://unsloth.ai/docs/basics/dynamic-3.0-ggufs)

## Choose a quantization

Start with a Dynamic 2-bit GGUF and choose a variant that fits your hardware. Unsloth's guide gives the following approximate memory planning ranges: system RAM plus VRAM, or unified memory on Apple silicon. Requirements vary with the exact quantization, context length, KV-cache format, and runtime configuration.

| Quantization | Approximate total memory |
| --- | --- |
| 2-bit | 300-315 GB |
| 3-bit | 365-390 GB |
| 4-bit | 470-500 GB |
| 6-bit | 670-700 GB |
| 8-bit | 870-950 GB |

BF16 language-model weights alone require approximately 1.63 TB before runtime overhead. Leave memory beyond the downloaded model size for the operating system, runtime, and KV cache. Longer contexts increase memory requirements. Offloading layers to system RAM reduces VRAM requirements; if memory pressure causes disk paging, generation can become much slower.

## Install Unsloth Desktop

[Download Unsloth](https://unsloth.ai/download) and complete the installer for your platform:

- [macOS](https://unsloth.ai/download/mac)
- [Windows](https://unsloth.ai/download/windows)
- [Linux and WSL](https://unsloth.ai/download/linux)

Alternatively, install from a terminal:

**macOS, Linux, WSL**

```bash
curl -fsSL https://unsloth.ai/install.sh | sh
```

**Windows PowerShell**

```powershell
irm https://unsloth.ai/install.ps1 | iex
```

Use a recent Unsloth release with Muse Spark Open support. See the [platform installation guides](https://unsloth.ai/docs/get-started/install) for additional setup instructions.

## Start the server

In Unsloth Desktop:

1. Launch Unsloth from your installed app shortcut.
2. Open **Model Hub**, search for "Muse Spark Open", and select the Unsloth GGUF repository linked above.
3. Download a quantization that fits your available memory. Start with Dynamic 2-bit; the CLI example below selects `UD-Q2_K_XL`.
4. Load the model from **Select model** on the Chat page and send a short message to confirm that it responds.
5. Open **Settings > API**, create an API key, and copy it. Use the endpoint URL shown in Unsloth when configuring your client.

Unsloth provides automatic per-model inference presets and lets you edit sampling settings, context length, and the chat template. Check the values against the [recommended settings](#recommended-settings) below when loading Muse Spark Open.

To load the model and start its API from the command line with the Unsloth CLI:

```bash
unsloth run \
  --model unsloth/Muse-Spark-Open-816B-A42B-GGUF:UD-Q2_K_XL \
  -H 127.0.0.1 \
  -p 8888 \
  --context-length 32768 \
  --temperature 1.0 \
  --top-p 1.0 \
  --min-p 0.0 \
  --reasoning-effort medium
```

The `:UD-Q2_K_XL` suffix selects the quantization. Unsloth starts the server, loads the UI, and prints the endpoint URL and API key. Keep this terminal running while sending requests.

This example starts with a 32,768-token context to limit KV-cache memory. Increase `--context-length` as your available memory allows. The `medium` reasoning level is a starting choice for interactive use; change it to another supported level as needed.

For a larger-memory configuration, select `:UD-Q4_K_XL` instead. Additional llama-server runtime flags let you adjust GPU layers, CPU threads, sampling, and networking; see the [Unsloth API guide](https://unsloth.ai/docs/basics/api).

## Recommended settings

| Setting | Recommended value |
| --- | --- |
| Temperature | `1.0` |
| Top-p | `1.0` |
| Min-p | `0.0` in the CLI example |
| Starting quantization | Dynamic 2-bit |
| Example serving context | `32768` tokens; adjust for available memory |
| Model maximum context | `1048576` tokens |
| Supported reasoning effort | `low`, `medium`, `high`, `xhigh` |

Use `--reasoning-effort` to pass the selected effort level to the model's chat template. The level takes effect only when the loaded template and backend support it. Choose `low` or `medium` for lighter reasoning, and `high` or `xhigh` for more demanding tasks. Muse Spark Open evaluations use `xhigh` for maximum reasoning strength.

The model's maximum context is not a recommended allocation for every machine. Context includes the input and generated tokens, and a larger KV cache needs additional memory. Allow enough output tokens for both reasoning and the final answer.

## Send requests

Unsloth exposes OpenAI-compatible `/v1/chat/completions` and `/v1/responses` endpoints, plus the Anthropic-compatible `/v1/messages` endpoint. Authenticate with your Unsloth API key. The examples below use Chat Completions.

In a second terminal, set the base URL and the API key copied from Unsloth. These shell examples use Bash on macOS, Linux, or WSL, and require Python 3 for model discovery.

```bash
export UNSLOTH_BASE_URL="http://127.0.0.1:8888"
export UNSLOTH_API_KEY="paste-your-unsloth-api-key-here"
```

Use the URL printed by your server if it differs. The `/v1/models` endpoint lists loaded and locally cached models. Select the loaded Muse Spark Open model's exact ID:

```bash
UNSLOTH_MODEL_ID="$(
  curl --fail-with-body "${UNSLOTH_BASE_URL}/v1/models" \
    -H "Authorization: Bearer ${UNSLOTH_API_KEY}" |
    python3 -c '
import json
import sys

models = [
    model
    for model in json.load(sys.stdin)["data"]
    if model.get("loaded") is True
    and "muse-spark-open" in model["id"].lower()
]
if len(models) != 1:
    raise SystemExit("Load exactly one Muse Spark Open model and retry.")
print(models[0]["id"])
'
)" && export UNSLOTH_MODEL_ID
```

Send a chat request. The Python example needs the OpenAI SDK (`python3 -m pip install -U openai`) and runs in the same environment as the exported client variables:

**curl**

```bash
curl --fail-with-body "${UNSLOTH_BASE_URL}/v1/chat/completions" \
  -H "Authorization: Bearer ${UNSLOTH_API_KEY}" \
  -H "Content-Type: application/json" \
  --data-binary @- <<JSON
{
  "model": "${UNSLOTH_MODEL_ID}",
  "messages": [
    {"role": "user", "content": "Explain mixture-of-experts models in three sentences."}
  ],
  "temperature": 1.0,
  "top_p": 1.0,
  "min_p": 0.0,
  "reasoning_effort": "medium",
  "enable_tools": false,
  "max_tokens": 8192,
  "stream": false
}
JSON
```

**Python (OpenAI SDK)**

```python
import os

from openai import OpenAI

client = OpenAI(
    base_url=os.environ["UNSLOTH_BASE_URL"].rstrip("/") + "/v1",
    api_key=os.environ["UNSLOTH_API_KEY"],
)

response = client.chat.completions.create(
    model=os.environ["UNSLOTH_MODEL_ID"],
    messages=[
        {"role": "user", "content": "Explain mixture-of-experts models in three sentences."}
    ],
    temperature=1.0,
    top_p=1.0,
    reasoning_effort="medium",
    max_tokens=8192,
    extra_body={"min_p": 0.0, "enable_tools": False},
)

print(response.choices[0].message.content)
```

`min_p` and `enable_tools` are Unsloth extensions to the Chat Completions format. Sending sampling values explicitly also applies them when using the desktop workflow. `enable_tools: false` keeps this request as text generation. For streaming, set `"stream": true` and add `--no-buffer` to curl.

## Check function calling

Send an OpenAI-style tool definition to request a structured function call:

```bash
curl --fail-with-body "${UNSLOTH_BASE_URL}/v1/chat/completions" \
  -H "Authorization: Bearer ${UNSLOTH_API_KEY}" \
  -H "Content-Type: application/json" \
  --data-binary @- <<JSON
{
  "model": "${UNSLOTH_MODEL_ID}",
  "messages": [
    {"role": "user", "content": "Use add_integers to calculate 17 plus 25."}
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "add_integers",
        "description": "Add two integers.",
        "parameters": {
          "type": "object",
          "properties": {
            "a": {"type": "integer"},
            "b": {"type": "integer"}
          },
          "required": ["a", "b"]
        }
      }
    }
  ],
  "tool_choice": "required",
  "temperature": 1.0,
  "top_p": 1.0,
  "min_p": 0.0,
  "reasoning_effort": "medium",
  "enable_tools": false,
  "max_tokens": 8192,
  "stream": false
}
JSON
```

Unsloth supports this client-side function-calling format; successful tool use also depends on the loaded model and its chat template. Check for a structured `choices[0].message.tool_calls` array. Your application executes client-defined functions: parse the arguments, run the function, then send the assistant's tool-call message and a `role: "tool"` result with the matching `tool_call_id` in the next request. This request disables Unsloth's built-in server-side tools so your application handles the defined function.

See the [curl and HTTP guide](https://unsloth.ai/docs/integrations/connect-curl-and-http-to-unsloth) and [Python SDK guide](https://unsloth.ai/docs/integrations/connect-python-sdk-to-unsloth) for additional endpoint examples.

## Connect a coding agent

Install your coding agent if it is not already available, load Muse Spark Open in Unsloth, and connect the agent:

```bash
unsloth start claude
```

You can also select the model while launching the agent:

```bash
unsloth start claude \
  --model unsloth/Muse-Spark-Open-816B-A42B-GGUF:UD-Q2_K_XL \
  --context-length 32768 \
  --temperature 1.0 \
  --top-p 1.0 \
  --min-p 0.0 \
  --reasoning-effort medium
```

On the default local address, passing `--model` starts a temporary server if Unsloth is not already running; that server stops when the agent exits. If Unsloth is already running, the launcher connects to it. Use `--temperature` for the Unsloth agent launcher. See [Unsloth Start](https://unsloth.ai/docs/integrations/unsloth-start) for supported agents and configuration options.

## Explore other workflows

- [Unsloth inference guide](https://unsloth.ai/docs/new/studio/chat): chat settings, web search, and Python or Bash execution.
- [Unsloth Desktop](https://unsloth.ai/docs/desktop): model downloads, training, and exporting workflows for supported models.
- [MCP integrations](https://unsloth.ai/docs/basics/mcp): connect additional tools.
- [llama.cpp](https://github.com/ggml-org/llama.cpp): run the GGUFs directly. Use a current build with Muse Spark Open support. Enable CUDA for NVIDIA GPUs; on Apple silicon, disable CUDA and use the default Metal backend.
- [llama-server documentation](https://github.com/ggml-org/llama.cpp/tree/master/tools/server): context sizing, GPU offload, threading, and sampling flags.
