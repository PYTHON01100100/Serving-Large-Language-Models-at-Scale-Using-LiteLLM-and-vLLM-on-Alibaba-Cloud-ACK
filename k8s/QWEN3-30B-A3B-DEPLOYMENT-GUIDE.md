# Qwen3-30B-A3B Deployment Guide

## Overview

This document explains the issues encountered while deploying the Qwen3-30B-A3B model on Kubernetes using vLLM, and how they were resolved.

---

## Background Concepts

### What is vLLM?

vLLM is a high-performance inference engine for Large Language Models (LLMs). Think of it as the "server" that loads a model into GPU memory and responds to chat/completion requests via an API.

### What is GPU Memory (VRAM)?

GPUs have their own dedicated memory called VRAM. When you run an LLM, the model's "weights" (the learned parameters that make the model intelligent) must be loaded into this VRAM. Our NVIDIA L20 GPU has ~45GB of VRAM.

### What is Quantization?

LLM weights are typically stored as numbers with high precision (like BF16 = 16 bits per number). **Quantization** reduces this precision to use less memory.

| Format | Bits per Weight | Memory Usage | Quality |
|--------|-----------------|--------------|---------|
| BF16 (default) | 16 bits | 100% (baseline) | Best |
| 8-bit (INT8) | 8 bits | ~50% | Very Good |
| 4-bit (INT4) | 4 bits | ~25% | Good |

**Analogy**: Imagine storing photos. RAW photos (BF16) have the best quality but take lots of space. JPEG (8-bit) compresses them with minimal visible quality loss. Heavily compressed JPEG (4-bit) saves even more space but you might notice some quality reduction.

### What are CUDA Graphs?

CUDA graphs are a GPU optimization technique that "records" a sequence of operations and replays them efficiently. This speeds up inference but requires extra memory during the initial "warmup" phase when the graphs are being captured.

---

## The Problem

### Initial Symptom

The pod kept crashing with `CrashLoopBackOff` status immediately after starting.

### Root Cause Analysis

**Issue 1: Model Too Large for GPU Memory**

```
torch.OutOfMemoryError: CUDA out of memory.
Tried to allocate 20.00 MiB. GPU has 44.17 GiB memory in use.
```

The Qwen3-30B-A3B model in BF16 format requires ~44GB of VRAM just to load the weights. Our L20 GPU has ~45GB total, leaving almost no room for:
- KV Cache (memory for storing conversation context)
- Intermediate computations during inference

**Issue 2: CUDA Graph Warmup OOM**

Even after reducing model size with quantization, the pod crashed during the "warmup" phase:

```
torch.OutOfMemoryError: CUDA out of memory. Tried to allocate 384.00 MiB.
GPU 0 has a total capacity of 44.53 GiB of which 359.94 MiB is free.
```

This happened because vLLM's default behavior captures CUDA graphs during startup, which temporarily requires extra memory for the quantization layer to decompress weights.

**Issue 3: Kubernetes Probe Timeouts**

The model takes ~2 minutes to load. Kubernetes health probes were timing out before the model finished loading, causing restarts.

**Issue 4: Shared Memory Exhaustion**

Rapid pod failures caused hundreds of failed pods with `UnexpectedAdmissionError` because each pod was requesting 16GB of shared memory from the node.

---

## The Solution

### Fix 1: Enable BitsAndBytes 8-bit Quantization

Added these flags to load the model in 8-bit quantized format:

```bash
--quantization bitsandbytes
--load-format bitsandbytes
```

**What this does**: Instead of loading the model at full 16-bit precision (44GB), it loads at 8-bit precision (16.8GB). The quantization happens *during* loading, not after, so we never need the full 44GB.

**Result**: Model memory reduced from ~44GB to 16.8GB

### Fix 2: Disable CUDA Graphs with Eager Mode

Added this flag:

```bash
--enforce-eager
```

**What this does**: Disables CUDA graph capture entirely. The model runs in "eager" mode where each operation executes immediately without pre-recording.

**Trade-off**: Slightly slower inference (~10-20%) but avoids the memory spike during warmup.

**Result**: No more OOM during engine initialization

### Fix 3: Increase Probe Delays

Changed the Kubernetes probe configuration:

```yaml
livenessProbe:
  initialDelaySeconds: 300  # Wait 5 minutes before first check
readinessProbe:
  initialDelaySeconds: 300  # Wait 5 minutes before first check
```

**What this does**: Gives the model enough time to fully load before Kubernetes starts checking if it's healthy.

### Fix 4: Reduce Shared Memory

```yaml
volumes:
  - name: shm
    emptyDir:
      medium: Memory
      sizeLimit: "2Gi"  # Reduced from 16Gi
```

**What this does**: Limits the shared memory allocation per pod, preventing node memory exhaustion when pods fail and retry.

---

## Final Configuration

### vLLM Command

```bash
vllm serve /mnt/OSSbuket/Qwen3-30B-A3B
  --host 0.0.0.0
  --port 8000
  --served-model-name Qwen3-30B-A3B
  --dtype bfloat16
  --max-model-len 4096
  --gpu-memory-utilization 0.90
  --quantization bitsandbytes
  --load-format bitsandbytes
  --trust-remote-code
  --enforce-eager
```

### Key Parameters Explained

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `--dtype bfloat16` | bfloat16 | Base data type for computations |
| `--max-model-len 4096` | 4096 tokens | Maximum context length per request |
| `--gpu-memory-utilization 0.90` | 90% | How much GPU memory vLLM can use |
| `--quantization bitsandbytes` | bitsandbytes | Use 8-bit quantization |
| `--load-format bitsandbytes` | bitsandbytes | Load weights in quantized format |
| `--enforce-eager` | (flag) | Disable CUDA graphs, use eager execution |
| `--trust-remote-code` | (flag) | Allow model's custom code to run |

---

## Performance Characteristics

After successful deployment:

| Metric | Value |
|--------|-------|
| Model Memory | 16.8 GiB |
| Available KV Cache | 21.96 GiB |
| Max Concurrent Requests (4k tokens each) | ~58 |
| Model Load Time | ~115 seconds |
| Total GPU Memory Used | ~40 GiB / 45 GiB |

---

## Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n your-namespace -l app=qwen3-30b-a3b
```

### Check Logs
```bash
kubectl logs -n your-namespace -l app=qwen3-30b-a3b --tail=100
```

### Check Previous Container Logs (after crash)
```bash
kubectl logs -n your-namespace -l app=qwen3-30b-a3b --previous --tail=200
```

### Test Health Endpoint
```bash
kubectl exec -n your-namespace deployment/qwen3-30b-a3b -- wget -qO- http://localhost:8000/health
```

### Common Issues

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| OOM during loading | Model too large | Use quantization |
| OOM during warmup | CUDA graph memory spike | Add `--enforce-eager` |
| Pod restarts before ready | Probe timeout | Increase `initialDelaySeconds` |
| UnexpectedAdmissionError | Node resource exhaustion | Delete deployment, reduce shared memory |

---

## Important Notes

1. **Always delete deployment before reapplying**: When making changes, always delete the existing deployment first to avoid resource conflicts:
   ```bash
   kubectl delete deployment qwen3-30b-a3b -n your-namespace
   kubectl apply -f qwen3-curent.yml
   ```

2. **Model quality**: 8-bit quantization has minimal impact on output quality for most use cases. The model's reasoning capabilities are preserved.

3. **Performance trade-off**: Using `--enforce-eager` disables CUDA graph optimizations. If you have a GPU with more memory in the future, you can remove this flag for better performance.

---

## Files

- **Deployment file**: `qwen3-curent.yml`
- **Node**: `me-central-1.10.3.1.163`
- **Namespace**: `your-namespace`
- **Service**: `qwen3-30b-a3b:8000`
