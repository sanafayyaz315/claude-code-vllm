# Claude Code with Self-Hosted Models via vLLM

Run [Claude Code](https://docs.anthropic.com/en/docs/claude-code) against self-hosted open-weight models served by [vLLM](https://docs.vllm.ai/), either standalone or on [Red Hat OpenShift AI (RHOAI)](https://www.redhat.com/en/technologies/cloud-computing/openshift/openshift-ai).

Claude Code is Anthropic's agentic coding CLI. It orchestrates multi-step coding workflows — reading files, editing code, running commands — using an LLM as the backend. vLLM v0.11.1+ exposes a native `/v1/messages` endpoint (Anthropic Messages API), which means Claude Code can connect to any vLLM-served model without a proxy or adapter.

## What's in This Repo

```
setup/
  deploy-model.md             # Deploy a model with vLLM (standalone or KServe on RHOAI)
  connect-claude-code.md      # Configure Claude Code to use the deployed model
  connect-claude-code.ipynb   # Same as above, as a runnable notebook

testing/
  evaluate-model.ipynb        # Automated test harness — runs 15 tasks via claude -p
  model-evaluation-template.md  # Blank template for recording evaluation results
  results/                    # Completed evaluation data
    qwen3-coder-evaluation.md   # Full evaluation report for Qwen3-Coder-30B-A3B
    baseline-claude-sonnet-4-6.json  # Baseline results (Claude Sonnet 4)
    qwen3-coder-2026-05-19.json     # Model results

spike-0-vllm-anthropic-api-integration.md  # Initial research spike
```

## Quick Start

### 1. Deploy a model

Choose one:

- **Standalone vLLM** — `pip install vllm` and run `vllm serve` locally with GPU(s)
- **KServe on RHOAI** — deploy via the RHOAI dashboard with the vLLM ServingRuntime

Both options are covered in [setup/deploy-model.md](setup/deploy-model.md).

### 2. Connect Claude Code

Set environment variables to point Claude Code at your vLLM endpoint, then launch `claude`. Full instructions in [setup/connect-claude-code.md](setup/connect-claude-code.md).

Standalone example:

```bash
export ANTHROPIC_BASE_URL=http://localhost:8000
export ANTHROPIC_API_KEY=sk-ant-placeholder-key
export CLAUDE_CODE_SKIP_AUTH_LOGIN=1

export ANTHROPIC_DEFAULT_OPUS_MODEL=qwen3-coder
export ANTHROPIC_DEFAULT_SONNET_MODEL=qwen3-coder
export ANTHROPIC_DEFAULT_HAIKU_MODEL=qwen3-coder
export ANTHROPIC_CUSTOM_MODEL_OPTION=qwen3-coder

claude
```

### 3. Evaluate the model

The evaluation notebook ([testing/evaluate-model.ipynb](testing/evaluate-model.ipynb)) runs 15 agentic coding tasks — from simple prompts to multi-file refactoring — against both a baseline (Claude Sonnet 4) and the self-hosted model using `claude -p` in non-interactive mode.

## Evaluated Model

**Qwen3-Coder-30B-A3B-Instruct** on 2x NVIDIA L40S via KServe, baselined against Claude Sonnet 4.

| | Baseline | Qwen3-Coder |
|---|---|---|
| Tasks passed | 15/15 | 13/15 |
| Total time | 666s | 3364s |

The model handles basic through intermediate tasks well (Q&A, code generation, bug fixing, test-driven development, API scaffolding, refactoring). It breaks down on sustained multi-turn sessions (30+ turns) and complex debugging workflows. Full results in [testing/results/qwen3-coder-evaluation.md](testing/results/qwen3-coder-evaluation.md).

## Key Findings

- vLLM's `/v1/messages` endpoint provides drop-in Anthropic API compatibility — no proxy needed
- `--enable-auto-tool-choice` and `--tool-call-parser` flags are required for Claude Code's tool-calling workflows
- `--served-model-name` must not contain `/` characters (Claude Code errors on slashes in model names)
- KServe requires `ANTHROPIC_AUTH_TOKEN` (Bearer auth); standalone vLLM uses `ANTHROPIC_API_KEY`
- All three `ANTHROPIC_DEFAULT_*_MODEL` vars must be set — Claude Code makes background Haiku calls that 404 if `ANTHROPIC_DEFAULT_HAIKU_MODEL` is missing

## Requirements

- vLLM v0.11.1+ (v0.17.1+ recommended for prefix cache fix)
- Claude Code CLI
- NVIDIA GPU(s) with sufficient VRAM for the model
- For RHOAI: OpenShift cluster with GPU nodes and `oc` CLI
