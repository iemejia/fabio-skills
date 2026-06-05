# Skill Eval Tests

This directory contains AI evaluation tests for the fabio skill using [promptfoo](https://github.com/promptfoo/promptfoo).

## How It Works

The eval tests verify that an LLM, when given the fabio skill instructions, produces correct CLI commands for common tasks. This catches regressions in instruction quality when skill content changes.

## Authentication

Uses **GitHub Models** (included with your Copilot subscription) - no separate OpenAI or Anthropic API keys needed. promptfoo's `github:` provider talks to `https://models.github.ai/inference` using your GitHub token.

Required: A GitHub PAT with `models:read` scope (same `COPILOT_PAT` used by the sync workflow).

## Running Locally

```bash
# Install promptfoo
npm install -g promptfoo

# Set your GitHub token (needs models:read scope)
export GITHUB_TOKEN=ghp_your_copilot_pat

# Run evals
promptfoo eval -c tests/promptfooconfig.yaml

# View results in browser
promptfoo view
```

## What's Tested

- **Command correctness**: Does the agent produce valid fabio CLI syntax?
- **PascalCase compliance**: Are API values like `Overwrite`, `Csv`, `Parquet` capitalized correctly?
- **Flag completeness**: Does the agent include required flags (`--workspace`, `--id`, etc.)?
- **Composability**: Does the agent use shell variables and piping patterns?
- **API quirk awareness**: Does the agent know about critical behaviors (LRO, token scoping, etc.)?

## Models Used

| Purpose | Model | Provider |
|---------|-------|----------|
| Agent simulation | `gpt-4o-mini` | GitHub Models |
| LLM-as-judge (rubric assertions) | `gpt-4o-mini` | GitHub Models |

You can swap to a stronger model (e.g., `github:openai/gpt-4o`) for higher quality judging at the cost of slower runs and higher rate limit consumption.

## Rate Limits

GitHub Models with Copilot Pro gives you 15 requests/minute and 150 requests/day for the "Low" tier models (includes gpt-4o-mini). Our test suite of ~20 cases fits comfortably within these limits.

## Cost

Zero additional cost beyond your existing Copilot subscription. All model inference is included.
