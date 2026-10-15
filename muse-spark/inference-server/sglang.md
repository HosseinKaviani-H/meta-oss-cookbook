# SGLang

Serve Muse Spark Open with [SGLang](https://docs.sglang.io/), an inference framework for low-latency, high-throughput serving on anything from a single GPU to a large distributed cluster.

## Links

[SGLang blog post on Muse Spark Open](https://www.lmsys.org/blog/2026-10-15-meta-muse-spark-open): the features SGLang supports for Muse Spark Open at launch.

[SGLang cookbook page for Muse Spark Open](https://docs.sglang.io/cookbook/autoregressive/Meta/MuseSparkOpen): detailed deployment options.

[DFlash2 draft model](https://huggingface.co/RadixArk/Muse-Spark-Open-DFlash2): a DFlash2 checkpoint trained for speculative decoding with Muse Spark Open.

## Install

Run Muse Spark Open with this Docker image:
```bash
docker pull lmsysorg/sglang:dev-muse-spark-open
```
For host setup, see the [SGLang installation documentation](https://docs.sglang.io/docs/get-started/install#method-3-using-docker).

## Start the server

To serve NVFP4 checkpoint:
```bash
sglang serve \
  --model-path meta-models/Muse-Spark-Open-816B-A42B-NVFP4 \
  --tp-size 4 \
  --ep-size 4 \
  --quantization modelopt_fp4 \
  --moe-runner-backend flashinfer_trtllm \
  --mem-fraction-static 0.8 \
  --host 0.0.0.0 \
  --port 30000 \
  --enable-layernorm-sp \
  --enable-megamoe-for-layernorm-sp \
  --speculative-algorithm DFLASH \
  --speculative-draft-model-path RadixArk/Muse-Spark-Open-DFlash2 \
  --speculative-draft-attention-backend fa4 \
  --speculative-num-draft-tokens 8 \
  --mm-feature-transport cpu \
  --enable-hierarchical-cache \
  --hicache-size 200 \
  --reasoning-parser muse \
  --tool-call-parser muse
```

To serve BF16 checkpoint:
```bash
sglang serve \
  --model-path meta-models/Muse-Spark-Open-816B-A42B \
  --tp-size 8 \
  --ep-size 8 \
  --dtype bfloat16 \
  --mem-fraction-static 0.85 \
  --host 0.0.0.0 \
  --port 30000 \
  --enable-layernorm-sp \
  --enable-megamoe-for-layernorm-sp \
  --speculative-algorithm DFLASH \
  --speculative-draft-model-path RadixArk/Muse-Spark-Open-DFlash2 \
  --speculative-draft-attention-backend fa4 \
  --speculative-num-draft-tokens 8 \
  --mm-feature-transport cpu \
  --enable-hierarchical-cache \
  --hicache-size 200 \
  --reasoning-parser muse \
  --tool-call-parser muse
```

## Recommended settings

For the best prefill performance on the NVFP4 checkpoint, run prefill with sequence parallelism and the MegaMoE kernel (`--enable-layernorm-sp --enable-megamoe-for-layernorm-sp`, which both server commands above already pass). Prefill is sharded across the four ranks and its MoE layers run on MegaMoE; decode stays on the TensorRT-LLM MoE kernel.

To tune it, add `--attention-backend trtllm_mha --page-size 128` to the NVFP4 server command above, and set the MegaMoE token limit before starting it:

```bash
export SGLANG_OPT_DEEPGEMM_MEGA_MOE_NUM_MAX_TOKENS_PER_RANK=8192
```

`SGLANG_OPT_DEEPGEMM_MEGA_MOE_NUM_MAX_TOKENS_PER_RANK` sets the maximum number of tokens that triggers MegaMoE. Raising it can improve prefill performance at larger batch sizes, but uses more device memory.

## Speculative decoding

The [DFlash2 draft model](https://huggingface.co/RadixArk/Muse-Spark-Open-DFlash2) proposes a block of eight draft tokens per step, and Muse Spark Open verifies them in a single pass. Both server commands above enable it with `--speculative-algorithm DFLASH` and the three `--speculative-*` flags after it. Remove those four flags to serve without speculative decoding.

## HiCache

HiCache offloads the KV cache to pinned host memory, so evicted prefixes are restored instead of recomputed. Both server commands above enable it with `--enable-hierarchical-cache --hicache-size 200`.

`--hicache-size` is the pinned host memory per tensor-parallel rank, in GB: the NVFP4 command (four ranks) pins 800 GB and the BF16 command (eight ranks) pins 1.6 TB. To size the host tier relative to the device KV pool instead, pass `--hicache-ratio N`. Remove both flags to serve without HiCache.

## Disaggregate prefill and decode

Prefill runs MegaMoE and sequence parallelism with HiCache. Decode runs the TensorRT-LLM MoE kernel with DFlash2. Run prefill and decode as two servers on four GPUs each, with a router in front that takes client requests:

- **B300:** both servers share one 8-GPU node, prefill on GPUs 0–3 and decode on GPUs 4–7.
- **GB300:** each server takes a full 4-GPU node, so prefill and decode run on separate nodes.

**1. Set the environment for your hardware.** Run the matching block before starting each server.

On B300, in the prefill terminal:

```bash
export MC_INTRANODE_NVLINK=true
export MC_TCP_ENABLE_CONNECTION_POOL=true
export SGLANG_MOONCAKE_CUSTOM_MEM_POOL=INTRA_NODE_NVLINK
export CUDA_VISIBLE_DEVICES=0,1,2,3
```

On B300, in the decode terminal:

```bash
export MC_INTRANODE_NVLINK=true
export MC_TCP_ENABLE_CONNECTION_POOL=true
export SGLANG_MOONCAKE_CUSTOM_MEM_POOL=INTRA_NODE_NVLINK
export CUDA_VISIBLE_DEVICES=4,5,6,7
```

On GB300, on both the prefill node and the decode node:

```bash
export NCCL_NVLS_ENABLE=1
export NCCL_MNNVL_ENABLE=1
export NCCL_CUMEM_ENABLE=1
export SGLANG_MOONCAKE_CUSTOM_MEM_POOL=NVLINK
export MC_FORCE_MNNVL=1
```

**2. Start the prefill server** (MegaMoE, sequence parallelism, and HiCache) on port 30001:

```bash
sglang serve \
  --model-path meta-models/Muse-Spark-Open-816B-A42B-NVFP4 \
  --quantization modelopt_fp4 \
  --moe-runner-backend flashinfer_trtllm \
  --tp-size 4 --ep-size 4 \
  --attention-backend trtllm_mha --page-size 128 \
  --mem-fraction-static 0.8 \
  --reasoning-parser muse --tool-call-parser muse \
  --enable-layernorm-sp --enable-megamoe-for-layernorm-sp \
  --enable-hierarchical-cache --hicache-size 200 \
  --speculative-algorithm DFLASH \
  --speculative-draft-model-path RadixArk/Muse-Spark-Open-DFlash2 \
  --speculative-draft-attention-backend fa4 \
  --speculative-num-draft-tokens 8 \
  --disaggregation-mode prefill \
  --disaggregation-transfer-backend mooncake \
  --host 0.0.0.0 --port 30001
```

**3. Start the decode server** (TensorRT-LLM MoE kernel and DFlash2) on port 30002, in a second terminal on B300 or on the decode node on GB300:

```bash
sglang serve \
  --model-path meta-models/Muse-Spark-Open-816B-A42B-NVFP4 \
  --quantization modelopt_fp4 \
  --moe-runner-backend flashinfer_trtllm \
  --tp-size 4 --ep-size 4 \
  --attention-backend trtllm_mha --page-size 128 \
  --mem-fraction-static 0.8 \
  --reasoning-parser muse --tool-call-parser muse \
  --speculative-algorithm DFLASH \
  --speculative-draft-model-path RadixArk/Muse-Spark-Open-DFlash2 \
  --speculative-draft-attention-backend fa4 \
  --speculative-num-draft-tokens 8 \
  --disaggregation-mode decode \
  --disaggregation-transfer-backend mooncake \
  --host 0.0.0.0 --port 30002
```

**4. Start the router** once both servers are up. Replace `<prefill-host>` and `<decode-host>` with addresses the router can reach; on B300, both are `127.0.0.1`.

```bash
sglang-router launch \
  --pd-disaggregation \
  --prefill http://<prefill-host>:30001 8998 \
  --decode http://<decode-host>:30002 \
  --host 0.0.0.0 --port 8000
```

`8998` after the prefill URL is the prefill server's bootstrap port. Send client requests to the router on port 8000: in the script under [Send requests](#send-requests), change the port from 30000 to 8000.

## Send requests

Here is a simple script for sending requests:
```python
from openai import OpenAI
client = OpenAI(
    base_url="http://127.0.0.1:30000/v1",
    api_key="null",
    timeout=600.0,
)
model = client.models.list().data[0].id
response = client.chat.completions.create(
    model=model,
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Briefly explain: what is SGLang?"},
    ],
    temperature=1.0,
    top_p=1.0,
    max_tokens=16384, # For setting the maximum number of output tokens
    reasoning_effort="high", # Can be one of low/medium/high/xhigh. xhigh by default.
)
choice = response.choices[0]
print("Reasoning:", getattr(choice.message, "reasoning_content", None))
print("Content:", choice.message.content)
print("Finish reason:", choice.finish_reason)
```

For other ways to send requests, such as curl or Python `requests`, see the [SGLang tutorial](https://docs.sglang.io/docs/basic_usage/send_request).

## More
To run Muse Spark Open on AMD GPUs or Intel Xeon, see [Hardware platforms](https://dev.meta.ai/docs/muse-spark-open/hardware-platforms). The [SGLang cookbook page](https://docs.sglang.io/cookbook/autoregressive/Meta/MuseSparkOpen) has more deployment options.

For more on SGLang, see the [SGLang website](https://docs.sglang.io/) and the [LMSYS blog](https://www.lmsys.org/blog/).
