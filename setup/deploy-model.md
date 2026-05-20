# Deploy a Model for Claude Code with vLLM

This guide covers two deployment options for serving a model via vLLM with Claude Code compatibility:

- **Option A: Standalone vLLM** — run vLLM locally or on any machine with GPUs
- **Option B: KServe on RHOAI** — deploy via Red Hat OpenShift AI's managed model serving

Both options use vLLM's native `/v1/messages` endpoint (Anthropic Messages API), which Claude Code connects to directly.

---

## vLLM Serving Flags for Claude Code

Regardless of deployment option, these vLLM flags are required for Claude Code compatibility:

| Flag | Purpose |
|------|---------|
| `--served-model-name <alias>` | Clean model alias — **must not contain `/` characters** (Claude Code errors on slashes) |
| `--enable-auto-tool-choice` | Required for Claude Code's tool-calling workflows |
| `--tool-call-parser <parser>` | Parser matching the model family (see table below) |

Common parsers:

| Model Family | Parser |
|-------------|--------|
| Qwen3-Coder | `qwen3_coder` |
| Devstral / Mistral | `mistral` |
| Llama | `llama3_json` |

---

## Option A: Standalone vLLM

### Prerequisites

- Python 3.9+
- NVIDIA GPU(s) with sufficient VRAM for the model
- CUDA toolkit installed

### Step 1: Install vLLM

```bash
pip install vllm
```

Verify the installation:

```bash
vllm --version
```

Minimum version: **v0.11.1** (adds `/v1/messages` support). Recommended: **v0.17.1+** (fixes prefix cache invalidation from Claude Code's attribution header).

### Step 2: Start the Server

```bash
vllm serve Qwen/Qwen3-Coder-30B-A3B-Instruct \
  --served-model-name qwen3-coder \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder \
  --tensor-parallel-size 2 \
  --max-model-len 131072
```

| Argument | Purpose |
|----------|---------|
| `Qwen/Qwen3-Coder-30B-A3B-Instruct` | HuggingFace model ID — downloaded automatically on first run |
| `--served-model-name qwen3-coder` | Clean alias (no `/` characters) |
| `--tensor-parallel-size 2` | Shard model across 2 GPUs — set to your GPU count |
| `--max-model-len 131072` | Cap context at 128K tokens to fit KV cache in GPU memory |

The first run downloads the model (~60GB) and compiles CUDA kernels. Subsequent starts are faster.

Wait for `Application startup complete` — the server is then ready on `http://localhost:8000`.

### Step 3: Verify the Endpoint

```bash
curl -X POST http://localhost:8000/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-coder",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "What is 2 + 2?"}]
  }'
```

Expected response:

```json
{
  "id": "chatcmpl-...",
  "type": "message",
  "role": "assistant",
  "content": [{"type": "text", "text": "2 + 2 = 4"}],
  "model": "qwen3-coder",
  "stop_reason": "end_turn",
  "usage": {"input_tokens": 16, "output_tokens": 8}
}
```

### Step 4: Connect Claude Code

Set the environment variables and launch Claude Code:

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

For the full list of environment variables and what they do, see [connect-claude-code.md](connect-claude-code.md).

### Standalone Gotchas

1. **Use `ANTHROPIC_API_KEY`, not `ANTHROPIC_AUTH_TOKEN`** — Standalone vLLM has no auth by default. Set `ANTHROPIC_API_KEY` to any non-empty value. The `sk-ant-` prefix is recommended — it triggers Claude Code's "Detected a custom API key" flow which avoids login prompts.

2. **First run downloads the full model** — ~60GB for Qwen3-Coder-30B. Set `HF_HOME` to control where HuggingFace caches the download.

3. **`--max-model-len` prevents OOM** — The full 256K context window requires more KV cache memory than most GPUs can spare after loading the model. Start with 131072 (128K) and adjust based on available VRAM.

4. **Attribution header slows inference on vLLM < 0.17.1** — Claude Code sends an attribution header that invalidates vLLM's prefix cache, causing ~90% slowdown. On vLLM < 0.17.1, disable it in `~/.claude/settings.json`:
   ```json
   {
     "env": {
       "CLAUDE_CODE_ATTRIBUTION_HEADER": "0"
     }
   }
   ```

---

## Option B: KServe on RHOAI

### Prerequisites

- Access to a RHOAI cluster with GPU nodes (NVIDIA GPUs)
- `oc` CLI installed and logged into the cluster
- A RHOAI project/namespace

### Step 1: Create the Model Deployment

In the RHOAI dashboard:

1. Navigate to **Model Serving** → **Deploy Model**
2. Configure the deployment:

| Setting | Value |
|---------|-------|
| **Serving Runtime** | vLLM NVIDIA GPU ServingRuntime for KServe |
| **Model URI** | `hf://Qwen/Qwen3-Coder-30B-A3B-Instruct` |
| **GPUs** | 2 |
| **Memory** | 96Gi |

3. Add the following **additional serving runtime arguments** — enter each on its own line, with **no leading or trailing spaces**:

```
--served-model-name=qwen3-coder
--tensor-parallel-size=2
--enable-auto-tool-choice
--tool-call-parser=qwen3_coder
--max-model-len=131072
```

| Argument | Purpose |
|----------|---------|
| `--served-model-name=qwen3-coder` | Clean alias for the model (no `/` characters allowed) |
| `--tensor-parallel-size=2` | Shard model across 2 GPUs — **must match the GPU count in resource limits** |
| `--enable-auto-tool-choice` | Required for Claude Code's tool-calling workflows |
| `--tool-call-parser=qwen3_coder` | Parser matching the Qwen3-Coder model family |
| `--max-model-len=131072` | Cap context window at 128K tokens to fit KV cache in available GPU memory |

4. Click **Deploy**

### Step 2: Verify GPU Allocation

The RHOAI dashboard may not set the GPU count correctly. Verify from your terminal:

```bash
oc get inferenceservice <name> -n <namespace> -o jsonpath='{.spec.predictor.model.resources}'
```

If the GPU count doesn't match `--tensor-parallel-size`, patch it:

```bash
oc patch inferenceservice <name> -n <namespace> --type='json' -p='[
  {"op": "replace", "path": "/spec/predictor/model/resources/limits/nvidia.com~1gpu", "value": "2"},
  {"op": "replace", "path": "/spec/predictor/model/resources/requests/nvidia.com~1gpu", "value": "2"}
]'
```

### Step 3: Wait for the Model to Download

The storage initializer downloads the model (~60GB). This takes 15–20 minutes depending on cluster bandwidth.

```bash
# Get the pod name
oc get pods -n <namespace>

# Watch download progress
oc logs <pod-name> -c storage-initializer -n <namespace> -f

# Check download size (run periodically)
oc exec <pod-name> -c storage-initializer -n <namespace> -- du -sh /mnt/models
```

### Step 4: Wait for vLLM to Start

After the download completes, vLLM loads the model into GPU memory (~6 minutes for 16 shards) and compiles CUDA kernels.

```bash
oc logs <pod-name> -c kserve-container -n <namespace> -f
```

Look for `Application startup complete` — that means the model is ready to serve requests.

### Step 5: Verify the Endpoint

Get the external endpoint:

```bash
oc get inferenceservice -n <namespace>
```

Test the `/v1/messages` endpoint:

```bash
curl -k -X POST https://<endpoint>/v1/messages \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -d '{
    "model": "qwen3-coder",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "What is 2 + 2?"}]
  }'
```

Expected response:

```json
{
  "id": "chatcmpl-...",
  "type": "message",
  "role": "assistant",
  "content": [{"type": "text", "text": "2 + 2 = 4"}],
  "model": "qwen3-coder",
  "stop_reason": "end_turn",
  "usage": {"input_tokens": 16, "output_tokens": 8}
}
```

### Step 6: Connect Claude Code

Set the environment variables and launch Claude Code:

```bash
export ANTHROPIC_BASE_URL=https://<endpoint>
export ANTHROPIC_AUTH_TOKEN=$(oc whoami -t)
export CLAUDE_CODE_SKIP_AUTH_LOGIN=1
export NODE_TLS_REJECT_UNAUTHORIZED=0

export ANTHROPIC_DEFAULT_OPUS_MODEL=qwen3-coder
export ANTHROPIC_DEFAULT_SONNET_MODEL=qwen3-coder
export ANTHROPIC_DEFAULT_HAIKU_MODEL=qwen3-coder

export ANTHROPIC_CUSTOM_MODEL_OPTION=qwen3-coder

claude
```

For the full list of environment variables and what they do, see [connect-claude-code.md](connect-claude-code.md).

### KServe Gotchas

1. **GPU count must match `--tensor-parallel-size`** — If the InferenceService resource limits specify 1 GPU but `--tensor-parallel-size=2`, vLLM crashes with "World size (2) is larger than the number of available GPUs (1)."

2. **RHOAI dashboard may not set GPU count correctly** — Always verify with `oc get inferenceservice` and patch if needed.

3. **Additional args must be separate entries** — The RHOAI dashboard concatenates args into a single string if entered with commas or spaces. Enter each argument on its own line with no leading/trailing spaces.

4. **`--max-model-len` needed for L40S GPUs** — The full 256K context window requires ~12GB KV cache per GPU, but only ~10GB is available after loading the model. Capping at 128K fits within the available memory. Without this, vLLM crashes with `torch.OutOfMemoryError`.

5. **Model URI must use `hf://` prefix** — Without it, the storage initializer fails with a crash loop.

6. **Model download is not cached across redeployments** — Each new InferenceService re-downloads the full model. Use a PVC for persistent storage to avoid repeated 60GB downloads.

7. **Use `ANTHROPIC_AUTH_TOKEN`, not `ANTHROPIC_API_KEY`** — KServe expects `Bearer` auth in the `Authorization` header. `ANTHROPIC_API_KEY` sends credentials as `x-api-key`, which KServe does not recognize.

---

## Useful `oc` Commands

```bash
# List all model deployments
oc get inferenceservice -n <namespace>

# Get pod status
oc get pods -n <namespace>

# View storage initializer logs (model download)
oc logs <pod-name> -c storage-initializer -n <namespace> -f

# View vLLM logs (model serving)
oc logs <pod-name> -c kserve-container -n <namespace> -f

# Check GPU allocation
oc describe pod <pod-name> -n <namespace> | grep -A 5 "nvidia.com/gpu"

# Check actual container args
oc get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[?(@.name=="kserve-container")].args}'

# Check InferenceService resource config
oc get inferenceservice <name> -n <namespace> -o jsonpath='{.spec.predictor.model.resources}'

# Delete a deployment
oc delete inferenceservice <name> -n <namespace>
```
