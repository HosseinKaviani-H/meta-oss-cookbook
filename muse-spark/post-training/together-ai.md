# Together AI

Adapt Muse Spark Open to your data with SFT and RL on Together AI's post-training service. Together AI manages the infrastructure and performance optimizations; you control the training loop. Muse Spark Open support on Together Post-Training is in preview, so the service and its limits can change.

## Links

- [GRPO quickstart](https://docs.together.ai/docs/post-training/quickstart-rl) and [SFT quickstart](https://docs.together.ai/docs/post-training/quickstart-sft) run ready-made recipes on Together Post-Training.
- [Post-training cookbook](https://github.com/togethercomputer/post-training-cookbook) has the recipes the quickstarts use.

## Get access

To get started with Together Post-Training:
1. Create a [Together AI account](https://api.together.ai/) and add credit to it.
2. Create an [API key](https://api.together.ai/settings/api-keys) and export it as `TOGETHER_API_KEY`.
3. Clone the [post-training cookbook](https://github.com/togethercomputer/post-training-cookbook), which installs the Together Python SDK.
4. Follow a quickstart from [Links](#links), with `meta-models/Muse-Spark-Open-816B-A42B` as the base model.

## Prepare your data

Read Together AI's guides on [preparing SFT data](https://docs.together.ai/docs/post-training/sft#prepare-your-data), [writing reward functions](https://docs.together.ai/docs/post-training/reward-functions), and [training in multi-turn environments](https://docs.together.ai/docs/post-training/multi-turn-environments).

## Use the tuned model

Learn how to deploy a post-trained model for inference with these guides:
1. [Deploy a checkpoint](https://docs.together.ai/docs/post-training/deployment)
2. [Multi-LoRA deployment](https://docs.together.ai/docs/post-training/multi-lora-deployment)
3. [Prepare for FP4 inference](https://docs.together.ai/docs/post-training/prepare-for-fp4-inference)
