# Spike 0: vLLM's Native Anthropic API Integration

**STRAT:** RHAIRFE-1820
**Status:** Complete

---

## Summary

vLLM v0.11.1+ natively supports the Anthropic Messages API via a `/v1/messages` endpoint. Claude Code speaks this API out of the box — no translation proxy or middleware needed. This spike validated that integration path and documented the configuration required to connect Claude Code to a vLLM-served model.

---

## Why No Middleware Is Needed

Claude Code communicates with its backend via the Anthropic Messages API (`/v1/messages`). The original assumption was that a translation layer would be needed to bridge Claude Code's Anthropic API calls to vLLM's OpenAI-compatible Chat Completions API (`/v1/chat/completions`).

However, vLLM added native Anthropic Messages API support in [**v0.11.1**](https://github.com/vllm-project/vllm/releases/tag/v0.11.1) via [PR #22627](https://github.com/vllm-project/vllm/pull/22627). This means vLLM now serves both APIs from the same endpoint:

- `/v1/messages` — Anthropic Messages API (what Claude Code uses)
- `/v1/chat/completions` — OpenAI Chat Completions API

Claude Code can connect directly to vLLM without any middleware.

---

## How the Integration Works

```
┌──────────────┐       ANTHROPIC_BASE_URL        ┌──────────────────────┐
│  Claude Code  │ ──── /v1/messages (POST) ────▶ │  vLLM on RHOAI       │
│  (CLI)        │ ◀─── Anthropic Messages API ── │  (KServe / ServingRT)│
└──────────────┘                                  └──────────────────────┘
```

1. Claude Code sends requests to `ANTHROPIC_BASE_URL/v1/messages` using the Anthropic Messages API format
2. vLLM receives the request, routes it to the served model, and returns a response in Anthropic Messages API format
3. Claude Code processes the response — including tool calls — and continues the agentic loop

Both streaming and non-streaming responses are supported.

---

## Claude Code Environment Variables

These environment variables configure Claude Code to use a vLLM endpoint instead of Anthropic's hosted API.

| Variable | Purpose | Required |
|----------|---------|----------|
| `ANTHROPIC_BASE_URL` | Points Claude Code at the vLLM endpoint (e.g., `http://localhost:8000` or `https://<kserve-endpoint>`) | Yes |
| `ANTHROPIC_API_KEY` | API key — set to any non-empty value if vLLM has no auth (e.g., `sk-ant-placeholder-key`) | Yes (one of API_KEY or AUTH_TOKEN) |
| `ANTHROPIC_AUTH_TOKEN` | Auth token sent as `Bearer` in the `Authorization` header — use this for KServe endpoints | Yes (for RHOAI/KServe) |
| `CLAUDE_CODE_SKIP_AUTH_LOGIN` | Set to `1` to skip Anthropic's login flow | Yes |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Model alias for Opus-tier requests | Yes |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Model alias for Sonnet-tier requests | Yes |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Model alias for Haiku-tier requests — Claude Code makes background Haiku calls that cause 404s if this is missing | Yes |
| `ANTHROPIC_CUSTOM_MODEL_OPTION` | Custom model name — Claude Code skips its built-in model name validation for this value | Yes |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_NAME` | Display name for the custom model in the CLI picker | Optional |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION` | Description for the custom model in the CLI picker | Optional |
| `CLAUDE_CODE_USE_VERTEX` | Set to `0` to disable Vertex AI routing (required if Vertex AI is configured in the environment) | Conditional |
| `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY` | Set to `1` to show actual model name in the `/model` picker instead of Opus/Sonnet/Haiku (requires v2.1.129+) | Optional |
| `NODE_TLS_REJECT_UNAUTHORIZED` | Set to `0` for self-signed TLS certificates (e.g., RHOAI routes) | Conditional |

**Key details:**

- All three `ANTHROPIC_DEFAULT_*_MODEL` vars must be set to the same model alias. Claude Code makes background Haiku calls — if `ANTHROPIC_DEFAULT_HAIKU_MODEL` is missing, those calls return 404.
- `ANTHROPIC_AUTH_TOKEN` vs `ANTHROPIC_API_KEY`: These control how Claude Code sends credentials to the backend.
  - `ANTHROPIC_API_KEY` is sent as `x-api-key` in the request header — this is the Anthropic API convention, but KServe does not recognize it.
  - `ANTHROPIC_AUTH_TOKEN` is sent as `Bearer <token>` in the `Authorization` header — this is what KServe routes expect.
  - **For RHOAI/KServe endpoints, always use `ANTHROPIC_AUTH_TOKEN`.** The value depends on where you're running Claude Code:
    - From your **local machine**: use your OpenShift token — `$(oc whoami -t)`
    - From a **RHOAI workbench**: use the pod's service account token — `$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)`
  - **For local vLLM** (no auth): use `ANTHROPIC_API_KEY` with a placeholder value (e.g., `sk-ant-placeholder-key`). Using the `sk-ant-` prefix is recommended — during workbench testing, this format triggered Claude Code's "Detected a custom API key" prompt.
  - Do not set both — Claude Code warns about auth conflicts.

---

## vLLM Serving Flags

These flags are required when starting vLLM to enable Claude Code compatibility.

| Flag | Purpose |
|------|---------|
| `--served-model-name <alias>` | Model alias — must not contain `/` characters (Claude Code errors on slashes in model names) |
| `--enable-auto-tool-choice` | Required for Claude Code's tool-calling workflows |
| `--tool-call-parser <parser>` | Parser matching the model family (e.g., `qwen3_coder` for Qwen3-Coder, `mistral` for Devstral) |

---

## Critical Gotchas

These were discovered during validation and apply to any Claude Code + vLLM setup.

1. **Attribution header cache invalidation** — On vLLM < 0.17.1, Claude Code's attribution header invalidates the KV prefix cache on every request, causing ~90% slowdown. On vLLM 0.17.1+, this is fixed. On older versions, disable it in `~/.claude/settings.json` (not via `export`):
   ```json
   {
     "env": {
       "CLAUDE_CODE_ATTRIBUTION_HEADER": "0"
     }
   }
   ```

2. **No `/` in model names** — Model names like `Qwen/Qwen3-Coder-30B-A3B` cause Claude Code errors. Always use `--served-model-name` to create a clean alias.

3. **Minimum 64K context window** — Claude Code's system prompt alone consumes ~16K tokens. Models with less than 64K context are unusable for real agentic coding.

4. **Model name validation** — Claude Code validates model names against a built-in list and rejects unknown names. `ANTHROPIC_CUSTOM_MODEL_OPTION` bypasses this.

5. **Vertex AI / Bedrock override** — If the environment has cloud provider config (Vertex AI, Bedrock, Foundry), Claude Code ignores `ANTHROPIC_BASE_URL` and routes to the cloud provider. Set `CLAUDE_CODE_USE_VERTEX=0` (or equivalent) to force the custom URL.

---

## Minimum vLLM Version

| Version | Support | Source |
|---------|---------|--------|
| **v0.11.1** | `/v1/messages` endpoint added — minimum for Claude Code integration | [Release notes](https://github.com/vllm-project/vllm/releases/tag/v0.11.1), [PR #22627](https://github.com/vllm-project/vllm/pull/22627) |
| **v0.17.1+** | Attribution header cache fix — recommended for production use | [Analysis](https://www.roborhythms.com/stop-claude-code-slowing-local-llm/) |

---

## References

- [vLLM v0.11.1 Release Notes](https://github.com/vllm-project/vllm/releases/tag/v0.11.1) — `/v1/messages` endpoint added
- [vLLM PR #22627](https://github.com/vllm-project/vllm/pull/22627) — Anthropic API implementation
- [vLLM Claude Code Integration Docs](https://docs.vllm.ai/en/stable/serving/integrations/claude_code/)
- [vLLM Tool Calling Docs](https://docs.vllm.ai/en/latest/features/tool_calling/)
- [Claude Code LLM Gateway Docs](https://code.claude.com/docs/en/llm-gateway) — environment variables and custom endpoint configuration
- [Attribution Header Cache Issue](https://www.roborhythms.com/stop-claude-code-slowing-local-llm/) — v0.17.1 fix analysis
- [Qwen3-Coder vLLM Recipe](https://docs.vllm.ai/projects/recipes/en/latest/Qwen/Qwen3-Coder-480B-A35B.html)
- [Claude Code on OpenShift with vLLM (Piotr Minkowski)](https://piotrminkowski.com/2026/02/27/claude-code-on-openshift-with-vllm-and-dev-spaces/)
