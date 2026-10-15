# Get the model

Download Muse Spark Open from Hugging Face and pick the checkpoint that matches your hardware. Everything runs offline once the weights land: there's no API key.

## Prerequisites

- Python 3.10 or later
- The Hugging Face CLI: `pip install -U huggingface_hub`
- Transformers, to check the tokenizer: `pip install -U "transformers>=5.19"`
- Enough local disk for the checkpoint you're downloading. See [Choose a checkpoint](#choose-a-checkpoint) for the footprint of each.

## Choose a checkpoint

Muse Spark Open has two checkpoint formats, and each ships as its own repository with the tokenizer, chat template, and processor config:

- BF16 is the full-precision reference.
- NVFP4 stores most weights in 4-bit and keeps some layers in BF16, which cuts the weight footprint to about a third.

| Checkpoint | Repository | Weight footprint | Serving recipes |
| --- | --- | --- | --- |
| BF16 | [`meta-models/Muse-Spark-Open-816B-A42B`](https://huggingface.co/meta-models/Muse-Spark-Open-816B-A42B) | About 1.65 TB | TP8/EP8 on 8 × B300 |
| NVFP4 | [`meta-models/Muse-Spark-Open-816B-A42B-NVFP4`](https://huggingface.co/meta-models/Muse-Spark-Open-816B-A42B-NVFP4) | About 537 GB | TP4/EP4 on 4 × B300 |

The weight footprint doesn't include the key-value cache, activations, or runtime overhead. The GPU count you need depends on accelerator memory, interconnect, context length, parallelism strategy, and the runtime.

## Download

The BF16 checkpoint:

```bash
hf download meta-models/Muse-Spark-Open-816B-A42B --local-dir ./muse-spark-open
```

The NVFP4 checkpoint:

```bash
hf download meta-models/Muse-Spark-Open-816B-A42B-NVFP4 --local-dir ./muse-spark-open-nvfp4
```

## Check the tokenizer

The repository ships the tokenizer, the chat template, `config.json`, and the processor config, so it carries everything a runtime needs to frame prompts. Load the tokenizer from the downloaded folder and render a prompt to confirm the download is usable. For the NVFP4 checkpoint, use `./muse-spark-open-nvfp4`:

```python
from transformers import AutoTokenizer

tok = AutoTokenizer.from_pretrained("./muse-spark-open")

messages = [{"role": "user", "content": "In one sentence, what is Muse Spark good at?"}]
print(tok.apply_chat_template(messages, add_generation_prompt=True, tokenize=False))
```

Each message in the rendered prompt starts with `<|start|>` and its role, its content follows `<|message|>`, and it ends with `<|eot|>`. For how to frame requests against a running server, see the [prompting guide](../prompting/README.md).

## Next steps

With the weights downloaded, read the [prompting guide](../prompting/README.md) to frame requests correctly, then [pick a runtime](../inference-server/README.md) and serve the model.
