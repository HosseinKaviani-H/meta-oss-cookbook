# Fireworks AI

Fireworks post-trains Muse Spark Open with serverless LoRA on a shared trainer. You write the training loop in Python. Fireworks runs the forward pass, backward pass, and optimizer on remote GPUs, then serves the current adapter for sampling in the same session. There is no trainer job or inference deployment to provision.

The base model is `accounts/fireworks/models/muse-spark-open-nvfp4` (Muse Spark Open 816B-A42B, NVFP4 open weights). Serverless training is LoRA only. The production pool supports LoRA rank up to 64 and sequences up to 262,144 tokens. Supervised fine-tuning, DPO, ORPO, distillation, and reinforcement learning all run on that pool. Full-parameter training uses dedicated GPUs.

## Links

- [Serverless training](https://docs.fireworks.ai/fine-tuning/training-api/serverless) covers setup, checkpoints, promotion, and pricing.
- [Fireworks cookbook](https://github.com/fw-ai/cookbook) has runnable SFT, DPO, and RL recipes, including the [serverless RL example](https://github.com/fw-ai/cookbook/tree/main/training/examples/serverless_rl).
- [Muse Spark Open weights](https://huggingface.co/meta-models/Muse-Spark-Open-816B-A42B-NVFP4) is the checkpoint and tokenizer the training client must match.

## Get access

1. Create a Fireworks account at [fireworks.ai](https://fireworks.ai).
2. Create an API key in the dashboard.
3. Install the training SDK and export the key:

```bash
pip install "fireworks-ai[training]"
export FIREWORKS_API_KEY="fw_..."
```

Serverless training does not require reserved GPUs. If `create_lora_training_client` reports that Muse Spark Open is not available on the account, contact Fireworks to enable it.

## Prepare your data

Keep every rendered example at or under 262,144 tokens. The open checkpoint's native context is 1,048,576 tokens. The serverless pool trains up to 262,144.

**Supervised fine-tuning.** One JSONL conversation per line, in the OpenAI chat format. A message with `"weight": 0` is left out of the loss. The cookbook `sft_loop` recipe reads this file.

```json
{"messages": [{"role": "system", "content": "You are a helpful assistant."}, {"role": "user", "content": "What is the capital of France?"}, {"role": "assistant", "content": "Paris."}]}
```

**Preference optimization.** Chosen and rejected responses for the same prompt. Use the cookbook DPO recipe for the row schema.

**Reinforcement learning.** Prompts plus a reward function you score locally. The serverless RL example uses rows of `{"messages": [...], "ground_truth": "{\"numbers\": [...], \"target\": N}"}`. `ground_truth` is a JSON string. Labeled assistant answers are not required.

Load the tokenizer from `meta-models/Muse-Spark-Open-816B-A42B-NVFP4`. It has to match the base model. Muse Spark Open is a mixture-of-experts model. Use the cookbook recipes so router replay stays enabled.

## Start a job

```python
import os
from fireworks.training.sdk import FiretitanServiceClient

service = FiretitanServiceClient(
    api_key=os.environ["FIREWORKS_API_KEY"],
    base_url="https://api.fireworks.ai/training/v1/serverless",
)

base_model = "accounts/fireworks/models/muse-spark-open-nvfp4"
training_client = service.create_lora_training_client(
    base_model=base_model,
    rank=64,
)
print(f"session={service.training_session_id} run={training_client.run_id}")
```

A session id that starts with `ts-` means the client attached to the pool. `rank` can be any positive integer up to 64.

For a full loop, clone the cookbook and start from the serverless RL example:

```bash
git clone https://github.com/fw-ai/cookbook
cd cookbook/training
pip install -e .
export FIREWORKS_API_KEY="fw_..."
python -m examples.serverless_rl.countdown_rl \
  --base-model accounts/fireworks/models/muse-spark-open-nvfp4 \
  --tokenizer-model meta-models/Muse-Spark-Open-816B-A42B-NVFP4
```

Each optimizer step is `forward_backward(...).result()` followed by `optim_step(...).result()`. Call `.result()` on every remote operation so failures are not dropped. If the pool is full, retry, or switch to dedicated training.

`save_state` writes a training checkpoint (adapter weights and optimizer state) for resume. `save_weights_for_sampler` writes a sampler checkpoint (adapter weights only) for sampling and deployment. A training checkpoint cannot be served. A sampler checkpoint cannot resume the optimizer.

## Use the tuned model

While the session is open, sample the current adapter directly. This continues the session from [Start a job](#start-a-job), using its `service` and `training_client`:

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("meta-models/Muse-Spark-Open-816B-A42B-NVFP4")
snapshot = training_client.save_weights_for_sampler("final").result().path
sampler = service.create_sampling_client(model_path=snapshot, tokenizer=tokenizer)
```

Keep the name passed to `save_weights_for_sampler` at 17 characters or fewer.

To serve after the session ends, promote the sampler checkpoint to a model, then deploy it. This also uses the `service` from [Start a job](#start-a-job). Promote before the session is deleted. After that, the checkpoints are no longer reachable through this API.

```python
import os
from datetime import datetime

from fireworks.training.sdk import FireworksClient

fw = FireworksClient(api_key=os.environ["FIREWORKS_API_KEY"])
rows = fw.list_training_session_checkpoints(service.training_session_name)
target = max(
    (row for row in rows if row.get("promotable")),
    key=lambda row: datetime.fromisoformat(row["createTime"].replace("Z", "+00:00")),
)
model = fw.promote_session_checkpoint(
    name=target["name"],
    output_model_id="muse-spark-my-lora",
    base_model="accounts/fireworks/models/muse-spark-open-nvfp4",
)
```

`output_model_id` is 1–63 characters: lowercase letters, digits, and hyphens.

Deploy the promoted LoRA with the Python SDK. `deployment_shape="default"` makes the server pick a validated shape for that model. No CLI is required.

```python
from fireworks import Fireworks

client = Fireworks()
client.deployments.create(
    base_model="accounts/<account-id>/models/muse-spark-my-lora",
    deployment_shape="default",
)
response = client.chat.completions.create(
    model="accounts/<account-id>/models/muse-spark-my-lora",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

A trained LoRA is served on a dedicated deployment. Fireworks does not serve a private trained adapter on per-token serverless inference.

## More

- Full-parameter training, and any run that needs its own GPUs for the whole job, uses [dedicated training](https://docs.fireworks.ai/fine-tuning/training-api/dedicated).
- Serverless training is billed per token on three meters: prefill, sample, and train. Cached prefill is discounted. Current rates are on the [serverless training](https://docs.fireworks.ai/fine-tuning/training-api/serverless) page.
- Checkpoint storage for serverless runs is included.
- The default concurrent-run quota is 8. Slots free when a run becomes terminal after the session expires.
