# Miles

[Miles](#links) is an open-source framework for large-scale reinforcement learning (RL) post-training. This guide runs GRPO on Muse Spark Open with a ready-made recipe on 64 GPUs, in one of two modes:

- **Math reasoning** (default): DAPO-Math-17k prompts, rewarded by matching the final `\boxed{}` answer.
- **Image-grounded math** (`--vlm`): Geometry3K diagrams and questions. The vision tower and its projection layers stay frozen; only the language model is trained.

## Links

- [Miles guide for Muse Spark Open](https://miles.radixark.com/docs/models/muse-spark/muse-spark-open)
- Miles: [radixark/miles, branch `muse-spark-open`](https://github.com/radixark/miles/tree/muse-spark-open)
- SGLang for Miles: [sgl-project/sglang, branch `sglang-miles-muse-spark-open`](https://github.com/sgl-project/sglang/tree/sglang-miles-muse-spark-open)
- Container image: `radixark/miles:muse-spark-open`
- Recipe: `scripts/run_muse_spark_64xh200.py` in the Miles repository
- Muse Spark Open checkpoint (BF16): [meta-models/Muse-Spark-Open-816B-A42B](https://huggingface.co/meta-models/Muse-Spark-Open-816B-A42B)

## Install

**Hardware.** The recipe needs 8 nodes with 8 NVIDIA H200 GPUs each, connected by InfiniBand. The full recipe is validated on H200. Blackwell GPUs (B200, B300) also run RL weight updates. You also need:

- storage visible to every node at `/root/models` and `/root/datasets` (or pass `--model-dir` and `--data-dir`) for the BF16 checkpoint and its Megatron copy (about 1.6 TB each) and the training data; the recipe downloads and converts the checkpoint there on its first run;
- local NVMe on every node at `/scratch` (or pass `--offload-train-disk-dir`) with about 1.2 TB free: the recipe streams Adam optimizer state there;
- to save the tuned model, storage visible to every node at `/root/shared_data` (or pass `--output-dir`); see [Use the tuned model](#use-the-tuned-model).

**1. Start the environment on every node.** Start one container per node with the GPUs, the InfiniBand devices, the host network, the shared storage and the local NVMe:

```bash
docker run -d --name miles --gpus all --network host --ipc host --shm-size 512g \
  --ulimit memlock=-1 --ulimit stack=67108864 --device /dev/infiniband \
  -v /path/to/shared/models:/root/models \
  -v /path/to/shared/datasets:/root/datasets \
  -v /path/to/shared/outputs:/root/shared_data \
  -v /path/to/local/nvme:/scratch \
  radixark/miles:muse-spark-open sleep infinity
docker exec -it miles bash
```

The image contains Miles and SGLang from the branches under [Links](#links), with Miles at `/root/miles`. Run the commands below inside the container.

**2. Start one Ray cluster across the 8 nodes.** On node 0, run `ray start --head --port 6379 --num-gpus 8`; on nodes 1–7, run `ray start --address <node-0-address>:6379 --num-gpus 8`. Check that `ray status` reports 64 GPUs.

## Prepare your data

By default the recipe downloads DAPO-Math-17k, or Geometry3K with `--vlm`, to `/root/datasets` (or pass `--data-dir`). You can train on your own dataset: download it as a JSONL file to `/root/datasets` and pass its path with `--extra-args "--prompt-data /root/datasets/train.jsonl"`. Each line is one prompt with these fields:

| Mode | Row fields |
|---|---|
| Math reasoning | `prompt` (a string or a list of chat messages), `label` |
| Image-grounded math (`--vlm`) | `messages` (the user turn, with one `<image>` placeholder per image), `images` (image paths), `label` |

`label` is the reference answer. The reward is 1 when the final `\boxed{}` answer in the response matches it and 0 otherwise, so ask for the final answer in `\boxed{}` in the prompt, as DAPO-Math-17k does.

## Start training

Run the recipe on node 0 from `/root/miles`. `MILES_SCRIPT_EXTERNAL_RAY=1` tells it to use the Ray cluster from step 2. To log to Weights & Biases, also export `WANDB_API_KEY`.

**Math reasoning:**

```bash
MILES_SCRIPT_EXTERNAL_RAY=1 python scripts/run_muse_spark_64xh200.py
```

**Image-grounded math:**

```bash
MILES_SCRIPT_EXTERNAL_RAY=1 python scripts/run_muse_spark_64xh200.py \
  --vlm --use-dynamic-sampling-filter --reasoning-effort medium --eval-interval 5
```

`--eval-interval 5` scores the Geometry3K test split (601 problems, 4 samples each) before training, every 5 steps, and after the last step.

What the recipe runs:

| Setting | Value |
|---|---|
| Algorithm | GRPO, no KL penalty, clip 0.2 / 0.28, rollout routing replay for the MoE layers |
| Batch | 32 prompts × 16 samples per step (512 rollouts), one update per step |
| Lengths | 4096-token responses, 8192-token context |
| Optimizer | Adam, learning rate 1e-6, state streamed to local disk; `--optimizer muon` switches to Muon, with its state in CPU memory |
| Training parallelism | tensor 8 × pipeline 8, expert 8, sequence parallelism, packed sequences in THD format, full recomputation |
| Rollout engines | 4 SGLang engines with 16 GPUs each, BF16 KV cache, up to 64 concurrent requests per engine |
| Reasoning effort | `xhigh` as default; the image-grounded math runs used `medium` to reduce response length |

Change these with the flags listed by `python scripts/run_muse_spark_64xh200.py --help` (for example `--num-rollout`, `--rollout-batch-size`, `--n-samples-per-prompt`, `--learning-rate`).

**What to expect.** Time per step on 64 H200 GPUs:

| Mode | Rollout | Training | Weight sync |
|---|---|---|---|
| Math reasoning | 2–5.5 min | 3–5.5 min | about 50 s |
| Image-grounded math | 8–14 min | about 5 min | about 50 s |

Image-grounded math rollouts take longer because `--use-dynamic-sampling-filter` discards prompts whose samples all get the same reward and keeps sampling until 32 prompts remain.

Healthy signs in the logs or in Weights & Biases:

- the mean absolute difference between rollout and training log-probabilities (`train/train_rollout_logprob_abs_diff`) stays around 0.007 (math reasoning) or 0.013 (image-grounded math);
- the reward is about 0.65–0.75 in the first steps; for math reasoning it reached 0.8–0.95 within 10 steps in RadixArk's runs, while the mean response length fell from about 2,600 to 800–1,900 tokens;
- with `--vlm`, Geometry3K accuracy starts near 0.87 and passed 0.93 within 5 steps in RadixArk's runs.

## Use the tuned model

Pass `--save-interval` when you start training, with a run ID so you know where the outputs go:

```bash
MILES_SCRIPT_EXTERNAL_RAY=1 python scripts/run_muse_spark_64xh200.py --save-interval 10 --run-id muse-dapo
```

The recipe saves the weights, without optimizer state, every 10 steps and after the last step to `/root/shared_data/muse-dapo/checkpoints`, then converts the last save into a Hugging Face checkpoint at `/root/shared_data/muse-dapo/hf` in the layout of `meta-models/Muse-Spark-Open-816B-A42B`. Each save and each conversion takes about 1.6 TB.

To convert another save, for example the one after step 10 (save directories count steps from 0, so it is `iter_0000009`):

```bash
python tools/convert_muse_spark_torch_dist_to_hf.py \
  --input-dir /root/shared_data/muse-dapo/checkpoints/iter_0000009 \
  --hf-checkpoint /root/models/Muse-Spark-Open-816B-A42B \
  --output-dir /root/shared_data/muse-dapo/hf-step-10
```

## More

- All recipe options: `python scripts/run_muse_spark_64xh200.py --help`.
- Miles documentation: [miles.radixark.com/docs](https://miles.radixark.com/docs)
