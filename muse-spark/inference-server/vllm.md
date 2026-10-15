# vLLM

Serve Muse Spark Open with [vLLM](https://docs.vllm.ai) behind an OpenAI-compatible endpoint.

## Links

- [vLLM recipe for Muse Spark Open](https://recipes.vllm.ai/meta-models/Muse-Spark-Open-816B-A42B)
- [vLLM launch blog for Muse Spark Open](https://vllm.ai/blog/2026-10-15-muse-spark-open)

## Prerequisites

- The downloaded weights. See [Get the model](../quickstart/README.md).
- NVIDIA or AMD GPUs. See [Choose a checkpoint](../quickstart/README.md#choose-a-checkpoint).

## Install

Prebuilt images are available for both vendors:

- NVIDIA: `vllm/vllm-openai:muse-spark-open`
- AMD: `vllm/vllm-openai-rocm:muse-spark-open`

Use one of the Docker images above to run Muse Spark Open.

## Start the server

NVFP4 on 4 × B300, with DSpark speculative decoding:

```bash
export VLLM_USE_RUST_FRONTEND=1

vllm serve meta-models/Muse-Spark-Open-816B-A42B-NVFP4 \
    --tensor-parallel-size 4 \
    --enable-expert-parallel \
    --max-model-len 1048576 \
    --reasoning-parser muse \
    --tool-call-parser muse \
    --enable-auto-tool-choice \
    --speculative-config '{"method":"dspark","model": "Inferact/Muse-Spark-Open-DSpark","num_speculative_tokens":7,"draft_sample_method":"probabilistic","rejection_sample_method":"block","enable_adaptive_verification":true}'
```

For BF16, serve `meta-models/Muse-Spark-Open-816B-A42B` with `--tensor-parallel-size 8` on 8 × B300 and keep the other flags.

`--max-model-len 1048576` serves the full 1,048,576-token context window. KV memory on the full-attention layers grows with context length, so if your workload needs less, lower it as described in [Context length](settings.md#context-length).

## Send a request

vLLM exposes an OpenAI-compatible endpoint on port 8000:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "meta-models/Muse-Spark-Open-816B-A42B-NVFP4",
    "messages": [{"role": "user", "content": "Refactor this module."}],
    "reasoning_effort": "xhigh"
  }'
```

Reasoning effort defaults to `xhigh`; set `reasoning_effort` to `low`, `medium`, `high`, or `xhigh`. See [Reasoning strength](settings.md#reasoning-strength).

Reasoning is returned separately from `content`, and tool calls arrive as a standard `tool_calls` array. Parsing happens server-side, so an OpenAI-compatible harness needs no model-specific handling.

## Send image, video, and audio

```bash
curl http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "meta-models/Muse-Spark-Open-816B-A42B-NVFP4",
    "messages": [{"role": "user", "content": [
      {"type": "image_url", "image_url": {"url": "https://example.com/screenshot.png"}},
      {"type": "video_url", "video_url": {"url": "https://example.com/demo.mp4"}},
      {"type": "audio_url", "audio_url": {"url": "https://example.com/clip.wav"}},
      {"type": "text",      "text": "What is happening here, and when?"}
    ]}]
  }'
```

## Stop tokens

Muse Spark Open needs to stop on `<|eot|>` and the end-of-text token. Never stop on `<|eom|>`, which ends a message while the turn continues. See [Stop tokens](settings.md#stop-tokens).
