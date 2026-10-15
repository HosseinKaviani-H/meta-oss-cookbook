# Meta open-weight model cookbook

Recipes for running, serving, and post-training Meta's open-weight Muse models on your own hardware. Each model has its own folder.

| Model | What it is | Go to |
|---|---|---|
| Muse Spark Open | Meta's 816B-parameter mixture-of-experts model with 42B active parameters, for agentic and coding work. Text, image, audio, and video in; text out. | [`muse-spark/`](muse-spark/) |
| Muse Glimmer | Meta's 30B open-weight multimodal model for local agentic work on a single GPU. | [`muse-glimmer/`](muse-glimmer/) |

## Contributing

Contributions welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) and the [Code of Conduct](CODE_OF_CONDUCT.md). Muse Glimmer recipes follow the shared template in [`muse-glimmer/assets/RECIPE_TEMPLATE.md`](muse-glimmer/assets/RECIPE_TEMPLATE.md).

## License

Please refer to the [License](LICENSE).
