# Connect Claude Code to a KServe Model on RHOAI

This guide walks through connecting Claude Code to a model already deployed via RHOAI's KServe model serving. For model deployment steps, see [deploy-model-kserve.md](deploy-model-kserve.md).

> **Prefer a runnable notebook?** These steps are also available as a Jupyter notebook: [connect-claude-code.ipynb](connect-claude-code.ipynb)

---

## Prerequisites

- A model deployed via KServe with the vLLM ServingRuntime (see [deploy-model-kserve.md](deploy-model-kserve.md))
- The KServe endpoint URL (RHOAI dashboard → Model Serving → your model → Inference endpoint)
- A RHOAI workbench with terminal access
- `oc` CLI installed and logged into the cluster (for granting permissions)

---

## Step 1: Grant Workbench Access to the KServe Endpoint

The workbench's service account needs permission to access the KServe endpoint. Without this, requests from the workbench return `Forbidden`.

Run the following from a **terminal with cluster admin access** (e.g., your local machine, not from inside the workbench — the workbench service account cannot grant itself roles):

```bash
oc adm policy add-role-to-user view system:serviceaccount:<namespace>:<workbench-sa> -n <namespace>
```

The workbench service account name typically matches the workbench name. For example:

```bash
oc adm policy add-role-to-user view system:serviceaccount:claude-code-vllm:claude-code-vllm -n claude-code-vllm
```

Verify the workbench can now reach the model by sending a test request from the **workbench terminal**. Replace `<kserve-endpoint>` with the external URL from the RHOAI dashboard (Model Serving → your model → Inference endpoint):

```bash
curl -k -X POST https://<kserve-endpoint>/v1/messages \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  -d '{
    "model": "qwen3-coder",
    "max_tokens": 50,
    "messages": [{"role": "user", "content": "Say hello"}]
  }'
```

You should get a valid JSON response from the model. If you get `Forbidden`, the role binding from the previous step hasn't taken effect — wait a moment and retry.

---

## Step 2: Install Claude Code

In the **workbench terminal**:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Verify the installation:

```bash
claude --version
```

## Step 3: Set Environment Variables

```bash
# Point Claude Code at the KServe endpoint
export ANTHROPIC_BASE_URL=<endpoint>
export ANTHROPIC_AUTH_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
export CLAUDE_CODE_SKIP_AUTH_LOGIN=1
export NODE_TLS_REJECT_UNAUTHORIZED=0

# Map all three model tiers to the self-hosted model
export ANTHROPIC_DEFAULT_OPUS_MODEL=qwen3-coder
export ANTHROPIC_DEFAULT_SONNET_MODEL=qwen3-coder
export ANTHROPIC_DEFAULT_HAIKU_MODEL=qwen3-coder

# Bypass Claude Code's built-in model name validation
export ANTHROPIC_CUSTOM_MODEL_OPTION=qwen3-coder
export ANTHROPIC_CUSTOM_MODEL_OPTION_NAME="Qwen3-Coder-30B-A3B on RHOAI"
export ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION="vLLM on RHOAI KServe"

# Disable Vertex AI / Bedrock routing if configured in the environment
export CLAUDE_CODE_USE_VERTEX=0
```

**Important notes:**

- Use `ANTHROPIC_AUTH_TOKEN` (not `ANTHROPIC_API_KEY`). Claude Code sends `ANTHROPIC_AUTH_TOKEN` as a `Bearer` token in the `Authorization` header, which is what KServe routes expect. `ANTHROPIC_API_KEY` is sent as `x-api-key`, which KServe does not recognize.
- Do **not** set both `ANTHROPIC_AUTH_TOKEN` and `ANTHROPIC_API_KEY` — Claude Code will warn about an auth conflict.
- `NODE_TLS_REJECT_UNAUTHORIZED=0` is needed because RHOAI routes use TLS with certificates that Node.js may not trust by default.
- All three `ANTHROPIC_DEFAULT_*_MODEL` vars must be set — Claude Code makes background Haiku calls that cause 404s if `ANTHROPIC_DEFAULT_HAIKU_MODEL` is missing.

## Step 4: Launch Claude Code

```bash
claude
```

## Step 5: Verify the Connection

Send a simple prompt to verify the connection:

```
What is 2 + 2?
```

Then try a prompt that tests tool usage (file creation):

```
Write a poem about winter and save it in poem.md
```

If you get responses and the file is created, the setup is complete: **Claude Code (workbench) → KServe → vLLM → Qwen3-Coder-30B-A3B**.

---

## Troubleshooting

### Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Retrying in Xs · attempt N/10` | Claude Code can't reach the endpoint | Check `ANTHROPIC_BASE_URL`, verify curl works from the workbench |
| `Unauthorized` | Wrong auth method | Use `ANTHROPIC_AUTH_TOKEN`, not `ANTHROPIC_API_KEY` |
| `Forbidden` | Service account lacks permissions | Run the `oc adm policy` command from Step 1 |
| `Auth conflict` warning | Both `ANTHROPIC_AUTH_TOKEN` and `ANTHROPIC_API_KEY` set | Unset `ANTHROPIC_API_KEY` |
| Login screen appears | `CLAUDE_CODE_SKIP_AUTH_LOGIN` not set or API key format not recognized | Ensure API key starts with `sk-ant-` if using `ANTHROPIC_API_KEY` |

For deployment-related issues (OOM crashes, GPU mismatches, args formatting), see [deploy-model-kserve.md](deploy-model-kserve.md).

---

## What Was Validated

| Component | Details |
|-----------|---------|
| **Model** | Qwen3-Coder-30B-A3B-Instruct |
| **Serving Runtime** | vLLM NVIDIA GPU ServingRuntime for KServe (v0.13.0+rhai11) |
| **GPUs** | 2x NVIDIA L40S (48GB each) |
| **GPU Memory per GPU** | ~28.5 GB for model weights (BF16) |
| **Tensor Parallelism** | 2 |
| **Max Context Length** | 131,072 tokens (128K) |
| **Attention Backend** | FLASH_ATTN |
| **Claude Code Version** | 2.1.142 |
| **API Compatibility** | Anthropic Messages API (`/v1/messages`) via vLLM |

### Validated Workflows

| Test | Result |
|------|--------|
| Simple prompt (`What is 2 + 2?`) | Pass — correct response |
| File creation (write a poem to `poem.md`) | Pass — used `Write` tool successfully |
| File read + analysis (count lines in `poem.md`) | Pass — read file and gave accurate count |
| Code generation (`fibonacci.py` with CLI args + error handling) | Pass — clean code, correct logic, proper error handling |
| File edit (add type hints and docstrings to existing file) | Pass — read, edited in place, added `typing` import, didn't break existing code |
| Bash tool (run `fibonacci.py` with n=10) | Pass — executed script, correct output |
| Multi-file project (calculator module + pytest tests + run) | Pass — created 2 files, ran pytest, all 5 tests passed |
| Long output handling (fibonacci n=100) | Partial — Bash tool produced full output but model truncated its summary |
