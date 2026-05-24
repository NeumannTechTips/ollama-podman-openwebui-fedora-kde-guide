# Advanced GPU Optimisation

This document covers GPU tuning, model selection strategy, performance
monitoring, and multi-model management for Ollama running on an NVIDIA RTX GPU
under Fedora 44 KDE Plasma with Podman.

---

## Table of Contents

- [Understanding GPU Layer Offloading](#understanding-gpu-layer-offloading)
- [Confirming Full GPU Utilisation](#confirming-full-gpu-utilisation)
- [VRAM Planning by Model Size](#vram-planning-by-model-size)
- [Quantisation Explained](#quantisation-explained)
- [Recommended Models by GPU](#recommended-models-by-gpu)
- [Ollama Environment Tuning](#ollama-environment-tuning)
- [Monitoring GPU Performance](#monitoring-gpu-performance)
- [Persistent Model Loading](#persistent-model-loading)
- [Running Multiple Models](#running-multiple-models)
- [Context Window Tuning](#context-window-tuning)

---

## Understanding GPU Layer Offloading

Ollama runs large language models by splitting them into layers. Each layer
can be offloaded to GPU VRAM for fast parallel computation, or left on the
CPU for slower serial computation. Full GPU offloading is the goal for
production-quality inference speed.

When a model loads, Ollama logs the offloading result:

```bash
podman logs -f ollama
```

A fully offloaded model produces output similar to:

```
llm server loading model
ggml_cuda_init: GGML_CUDA_FORCE_MMQ: no
ggml_cuda_init: CUDA_USE_TENSOR_CORES: yes
llama_model_load: offloading 32 repeating layers to GPU
llama_model_load: offloading non-repeating layers to GPU
llama_model_load: offloaded 33/33 layers to GPU
llama_model_load: VRAM used: 7842 MiB
```

If the log shows `offloaded 0/33 layers to GPU`, the GPU is not accessible
to the container. Refer to the [Troubleshooting Guide](troubleshooting.md).

If the log shows partial offloading such as `offloaded 18/33 layers to GPU`,
the model is too large for the available VRAM. The remainder runs on CPU,
which significantly reduces inference speed.

---

## Confirming Full GPU Utilisation

Run `nvidia-smi` while a model is actively generating a response:

```bash
watch -n 1 nvidia-smi
```

During active inference you should observe:

- **GPU-Util:** 70% to 99%
- **Memory-Usage:** Near the VRAM capacity of the loaded model
- **Power usage:** Approaching the card's TDP

If GPU-Util stays at 0% during generation, the model is running entirely on
the CPU. Check the Ollama logs and the CDI configuration.

---

## VRAM Planning by Model Size

Model VRAM requirements depend on parameter count and quantisation level.
The figures below are approximate for common quantisation formats (Q4\_K\_M).

| Parameters | Approx. VRAM (Q4\_K\_M) | Approx. VRAM (FP16) |
|-----------|-------------------------|---------------------|
| 3B | 2.5 GB | 6 GB |
| 7B to 8B | 4.5 to 5 GB | 14 to 16 GB |
| 13B | 7.5 GB | 26 GB |
| 14B | 8.5 GB | 28 GB |
| 32B | 18 GB | 64 GB |
| 70B | 38 to 40 GB | 140 GB |

> **Rule of thumb:** For a given parameter count at Q4\_K\_M quantisation,
> multiply the parameter count (in billions) by approximately 0.55 to estimate
> the VRAM requirement in GB. Add 1 GB for the KV cache and model overhead.

---

## Quantisation Explained

Quantisation reduces a model's numerical precision to lower VRAM requirements
at the cost of a small reduction in output quality. The most relevant formats
for Ollama users are:

| Format | Bits | Quality | VRAM |
|--------|------|---------|------|
| FP16 | 16 | Reference quality | Highest |
| Q8\_0 | 8 | Near-lossless | High |
| Q6\_K | 6 | Excellent | Moderate-high |
| Q5\_K\_M | 5 | Very good | Moderate |
| Q4\_K\_M | 4 | Good (recommended default) | Low-moderate |
| Q3\_K\_M | 3 | Acceptable for smaller models | Low |
| Q2\_K | 2 | Noticeable quality loss | Lowest |

For most engineers running on an RTX 3060 12 GB, **Q4\_K\_M** provides the
best balance of quality and VRAM efficiency. Use Q5\_K\_M or Q6\_K if VRAM
permits.

To pull a specific quantisation variant:

```bash
podman exec -it ollama ollama pull llama3.1:8b-instruct-q4_K_M
podman exec -it ollama ollama pull qwen3:8b-q5_K_M
```

---

## Recommended Models by GPU

### RTX 3060 12 GB

| Use Case | Recommended Model | Pull Command |
|----------|------------------|--------------|
| General chat | `llama3.1:8b` | `ollama pull llama3.1:8b` |
| Coding assistant | `qwen3:8b` | `ollama pull qwen3:8b` |
| Fast lightweight | `llama3.2:3b` | `ollama pull llama3.2:3b` |
| Reasoning | `deepseek-r1:8b` | `ollama pull deepseek-r1:8b` |

### RTX 4070 12 GB / 4070 Ti 12 GB

Same recommendations as RTX 3060 12 GB, with the added option of 13B models
at Q4\_K\_M.

### RTX 4080 16 GB

| Use Case | Recommended Model | Pull Command |
|----------|------------------|--------------|
| General chat | `llama3.1:13b` | `ollama pull llama3.1:13b` |
| Coding assistant | `qwen3:14b` | `ollama pull qwen3:14b` |
| Reasoning | `deepseek-r1:14b` | `ollama pull deepseek-r1:14b` |

### RTX 4090 24 GB / RTX 3090 24 GB

| Use Case | Recommended Model | Pull Command |
|----------|------------------|--------------|
| General chat | `llama3.1:33b-q4_K_M` | `ollama pull llama3.1:33b-instruct-q4_K_M` |
| Coding assistant | `qwen3:32b-q4_K_M` | `ollama pull qwen3:32b-q4_K_M` |
| Reasoning | `deepseek-r1:32b` | `ollama pull deepseek-r1:32b` |

---

## Ollama Environment Tuning

The following environment variables can be added to the `ollama` service in
`compose.yml` to tune behaviour:

```yaml
environment:
  - OLLAMA_HOST=0.0.0.0:11434
  - OLLAMA_KEEP_ALIVE=24h
  - OLLAMA_MAX_LOADED_MODELS=1
  - OLLAMA_NUM_PARALLEL=1
  - OLLAMA_FLASH_ATTENTION=1
```

| Variable | Description |
|----------|-------------|
| `OLLAMA_KEEP_ALIVE` | How long to keep a model loaded in VRAM after the last request. Set to `24h` to avoid cold-start latency. Set to `0` to unload immediately and free VRAM. |
| `OLLAMA_MAX_LOADED_MODELS` | Maximum number of models simultaneously loaded in VRAM. Set to `1` on 12 GB cards to prevent VRAM exhaustion. |
| `OLLAMA_NUM_PARALLEL` | Number of parallel inference requests. Set to `1` on single-GPU consumer setups. |
| `OLLAMA_FLASH_ATTENTION` | Enables Flash Attention 2 for supported models. Reduces VRAM usage and increases throughput on Turing (RTX 20xx) and newer GPUs. Set to `1` to enable. |

After modifying `compose.yml`, recreate the container:

```bash
cd ~/ollama-podman
podman-compose up -d
```

---

## Monitoring GPU Performance

### Real-time GPU monitoring

```bash
# Monitor GPU utilisation, temperature, and memory every second
watch -n 1 nvidia-smi

# Show only memory and utilisation
watch -n 1 "nvidia-smi --query-gpu=name,temperature.gpu,utilization.gpu,memory.used,memory.free --format=csv,noheader"
```

### Monitor running inference processes

```bash
nvidia-smi --query-compute-apps=pid,process_name,used_gpu_memory --format=csv
```

### Container resource usage

```bash
# Live container stats (CPU, memory, network)
podman stats

# One-time snapshot without streaming
podman stats --no-stream
```

### Inference speed from logs

```bash
# Tokens per second are logged by Ollama during generation
podman logs ollama 2>&1 | grep "eval rate"
```

A well-optimised RTX 3060 12 GB running a Q4\_K\_M 8B model typically
produces 40 to 80 tokens per second.

---

## Persistent Model Loading

By default, Ollama unloads a model from VRAM after 5 minutes of inactivity.
For a desktop workstation where you want the model immediately available:

```yaml
# In compose.yml under the ollama service environment block
- OLLAMA_KEEP_ALIVE=24h
```

To unload a model immediately and free VRAM without stopping the container:

```bash
podman exec -it ollama ollama stop llama3.1:8b
```

---

## Running Multiple Models

On GPUs with 24 GB or more of VRAM, it is practical to keep two smaller
models loaded simultaneously.

Set `OLLAMA_MAX_LOADED_MODELS=2` in `compose.yml` and pull two models:

```bash
podman exec -it ollama ollama pull llama3.1:8b
podman exec -it ollama ollama pull qwen3:8b
```

Monitor VRAM to confirm both fit comfortably:

```bash
nvidia-smi --query-gpu=memory.used,memory.free --format=csv,noheader
```

On a 12 GB card, loading two 8B Q4\_K\_M models simultaneously
(approximately 9 to 10 GB combined) leaves very little headroom and is not
recommended.

---

## Context Window Tuning

The default context window for most Ollama models is 2048 tokens. For longer
conversations or document analysis, you can increase this via the Open WebUI
Advanced Parameters panel or via the Ollama API.

Be aware that a larger context window consumes proportionally more VRAM for
the KV cache. On a 12 GB card running an 8B model at Q4\_K\_M:

| Context (tokens) | Approx. additional VRAM |
|-----------------|------------------------|
| 2048 (default) | Baseline |
| 4096 | +0.5 GB |
| 8192 | +1 GB |
| 16384 | +2 GB |
| 32768 | +4 GB |

Exceeding available VRAM will cause the model to partially offload to CPU,
reducing inference speed significantly.

---

*For common errors and fixes, refer to the [Troubleshooting Guide](troubleshooting.md).*
