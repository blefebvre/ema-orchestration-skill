# ema-orchestration-skill

A SLICC skill that orchestrates the [Experience Modernization Agent](https://aemcoder.adobe.io/) to migrate a URL to AEM Edge Delivery Services.

## Install

```
upskill blefebvre/ema-orchestration-skill
```

## Usage

> "Migrate the page `<URL>` to Edge Delivery using EMA"

The skill handles pre-flight checks (EMA login, GitHub connection), drives the migration UI, performs visual QA, and issues iterative refinement prompts until the result is pixel-perfect.
