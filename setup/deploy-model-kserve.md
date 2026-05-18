# Deploy a Model via RHOAI Model Serving (KServe)

This guide walks through deploying a model on RHOAI using KServe with the vLLM NVIDIA GPU ServingRuntime.

---

## Prerequisites

- Access to a RHOAI cluster with GPU nodes (NVIDIA GPUs)
- `oc` CLI installed and logged into the cluster
- A RHOAI project/namespace

---

## Step 1: Create the Model Deployment

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

## Step 2: Verify GPU Allocation

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

## Step 3: Wait for the Model to Download

The storage initializer downloads the model (~60GB). This takes 15–20 minutes depending on cluster bandwidth.

```bash
# Get the pod name
oc get pods -n <namespace>

# Watch download progress
oc logs <pod-name> -c storage-initializer -n <namespace> -f

# Check download size (run periodically)
oc exec <pod-name> -c storage-initializer -n <namespace> -- du -sh /mnt/models
```

## Step 4: Wait for vLLM to Start

After the download completes, vLLM loads the model into GPU memory (~6 minutes for 16 shards) and compiles CUDA kernels.

```bash
oc logs <pod-name> -c kserve-container -n <namespace> -f
```

Look for `Application startup complete` — that means the model is ready to serve requests.

## Step 5: Verify the Endpoint

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

---

## Deployment Gotchas

1. **GPU count must match `--tensor-parallel-size`** — If the InferenceService resource limits specify 1 GPU but `--tensor-parallel-size=2`, vLLM crashes with "World size (2) is larger than the number of available GPUs (1)."

2. **RHOAI dashboard may not set GPU count correctly** — Always verify with `oc get inferenceservice` and patch if needed.

3. **Additional args must be separate entries** — The RHOAI dashboard concatenates args into a single string if entered with commas or spaces. Enter each argument on its own line with no leading/trailing spaces.

4. **`--max-model-len` needed for L40S GPUs** — The full 256K context window requires ~12GB KV cache per GPU, but only ~10GB is available after loading the model. Capping at 128K fits within the available memory. Without this, vLLM crashes with `torch.OutOfMemoryError`.

5. **Model URI must use `hf://` prefix** — Without it, the storage initializer fails with a crash loop.

6. **Model download is not cached across redeployments** — Each new InferenceService re-downloads the full model. Use a PVC for persistent storage to avoid repeated 60GB downloads.

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
