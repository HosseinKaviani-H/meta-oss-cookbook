# llama.cpp

Portable inference across Metal, CUDA, ROCm, Vulkan, and CPU. llama.cpp and the Llama app support Muse Spark Open.

## Install

Install a prebuilt llama binary for your platform:

```bash
curl -LsSf https://llama.app/install.sh | sh
```

## Serve

`llama serve -hf` downloads, caches, and serves a model in one command. Muse Spark Open supports up to 1,048,576 tokens of context.

```bash
llama serve -hf ggml-org/Muse-Spark-Open-816B-A42B-GGUF --chat-template-kwargs '{"reasoning_effort":"low"}'
```

To serve a specific quantization, add its suffix. ggml-org ships three quantizations for this release: Q4_K_S, Q8_0, and Q2_K.

```bash
llama serve -hf ggml-org/Muse-Spark-Open-816B-A42B-GGUF:Q4_K_S
llama serve -hf ggml-org/Muse-Spark-Open-816B-A42B-GGUF:Q8_0
llama serve -hf ggml-org/Muse-Spark-Open-816B-A42B-GGUF:Q2_K
```

The server exposes an OpenAI-compatible endpoint. Open `http://localhost:8080` to chat with Muse Spark Open in the Llama WebUI, which supports images, audio, and Model Context Protocol (MCP) tools.

Useful flags:

| Flag | Why |
| --- | --- |
| `-a muse-spark-open` | API alias. By default, this is the checkpoint path. |
| `--chat-template-kwargs` | Sets `reasoning_effort`, see below. Template default is `xhigh`. |
| `-np N` | Concurrent slots. The context length is divided across the slots, so long-context agents should use `-np 1`. |
| `--api-key` | Optional key to gate endpoints. `/health` and `/v1/models` still answer without it. |

Smoke test (`--noproxy` ensures the loopback request connects directly):

```bash
curl -s --noproxy '*' http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" -H "Authorization: Bearer <your-key>" \
  -d '{"model":"muse-spark-open","messages":[{"role":"user","content":"What is 17 * 23?"}],"temperature":1.0}'
```

## Control reasoning

The chat template reads `reasoning_effort` and defaults to `xhigh`.

Set it server-wide with `--chat-template-kwargs`, or per request:

```json
{"model":"muse-spark-open","messages":[...],"chat_template_kwargs":{"reasoning_effort":"low"}}
```

The model supports four levels: `low`, `medium`, `high`, `xhigh`.

Reasoning can't be turned off. The chat template rejects `"reasoning_effort": "none"`, and `--reasoning off` leaves the output unchanged. Set `reasoning_effort` to `low` to spend fewer thinking tokens.

## Vision

You can upload images through the Web UI. For applications that use the server endpoint, send a remote URL, a `data:image/...;base64,...` URI, or a local path:

```bash
curl -s http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" -H "Authorization: Bearer <your-key>" \
  -d '{"model":"muse-spark-open","messages":[{"role":"user","content":[
        {"type":"text","text":"What is in this image?"},
        {"type":"image_url","image_url":{"url":"data:image/png;base64,..."}}
      ]}],"temperature":1.0}'
```

## Audio

Similarly to vision, you can either upload through the Web UI or send base64 encoded files to the endpoint. Audio follows the OpenAI `input_audio` shape: raw base64 in `data`, not a data URL, and the file type (`wav` or `mp3`) in `format`.

```bash
curl -s http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" -H "Authorization: Bearer <your-key>" \
  -d '{"model":"muse-spark-open","messages":[{"role":"user","content":[
        {"type":"text","text":"Transcribe this audio."},
        {"type":"input_audio","input_audio":{"data":"<base64>","format":"wav"}}
      ]}],"temperature":1.0}'
```

## CLI

You can also use `llama cli` to interact with the model through CLI.

```bash
llama cli -hf ggml-org/Muse-Spark-Open-816B-A42B-GGUF -c 32768
```

After you launch `llama cli`, use the `/image` command to pass in images. You can then enter a text prompt.

```bash
> /image image.png
Loaded media from 'image.png'

> describe this image
```

Similarly, to pass audio:

```bash
> /audio /path/example_audio.mp3

Loaded media from '/path/example_audio.mp3'

> transcribe this audio
```

## Verify tool calling

```bash
curl -s http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" -H "Authorization: Bearer <your-key>" \
  -d '{"model":"muse-spark-open","messages":[{"role":"user","content":"What is the weather in Paris in celsius? Use the tool."}],
       "tools":[{"type":"function","function":{
         "name":"get_weather","description":"Get current weather for a city.",
         "parameters":{"type":"object","properties":{
           "city":{"type":"string"},"units":{"type":"string","enum":["celsius","fahrenheit"]}},
           "required":["city"]}}}],
       "tool_choice":"auto","temperature":1.0}'
```

The command above returns finish_reason: `"tool_calls"`, a `tool_calls` array containing `get_weather(city="Paris", units="celsius")`, and reasoning in `reasoning_content`. Tool-call parsing occurs server-side, so OpenAI-compatible clients require no Muse Spark Open–specific logic.

## Context per slot

Check `n_ctx_slot` in the startup log to determine the per-generation context limit. Don't read the limit from `-c`: it sets the total context length, shared across slots. When a generation exceeds its slot limit, the server may return neither an answer nor an error, making batch and evaluation failures difficult to diagnose.

To increase concurrency while retaining the same per-slot limit, increase `-c` in proportion to `-np`:

```bash
  -c 524288 -np 2        # 262144 per slot and 2-way concurrent
```
