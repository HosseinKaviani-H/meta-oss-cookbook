# NVIDIA Dynamo

[NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo) provides an OpenAI-compatible frontend, reasoning and tool-call parsing, disaggregated serving, and KV-aware routing.

## Links

- [Dynamo documentation](https://docs.nvidia.com/dynamo/latest/)
- [Muse Spark Open Dynamo recipe](https://docs.nvidia.com/dynamo/recipes/Muse-Spark-Open)

## Install

See the [Dynamo installation guide](https://docs.nvidia.com/dynamo/cli/installation/install-dynamo). Use the Muse Spark Open-enabled image and checkpoint specified in the model recipe, including the latest tokenizer, processor, and chat template.

## Start the server

Follow the model recipe for launch commands and backend settings. The [Dynamo quickstart](https://docs.nvidia.com/dynamo/cli/getting-started/quickstart) explains how the frontend and model worker run together.

## Recommended settings

See the [Dynamo recipes](https://github.com/ai-dynamo/dynamo/tree/main/recipes) for recommended deployment settings.

## Send requests

See the request examples in the [Dynamo quickstart](https://docs.nvidia.com/dynamo/cli/getting-started/quickstart) and model recipe. Chat completions separate `reasoning_content` from `content` and return structured `tool_calls` when tools are supplied.

## More

- [Dynamo recipe catalog](https://github.com/ai-dynamo/dynamo/tree/main/recipes)
- [Disaggregated serving](https://docs.nvidia.com/dynamo/cli/disaggregated-serving/overview)
- [KV-aware routing](https://docs.nvidia.com/dynamo/cli/kv-aware-routing/overview)
