+++
title = "Distributed Training Infrastructure: Scaling to GPT-2"
description = "Engineering high-performance training pipelines for Large Language Models using DDP, gradient accumulation, and hardware-specific optimizations."
date = 2024-03-26
math = true
+++

Scaling from toy-sized Transformers to Large Language Models (LLMs) like GPT-2 is a fundamental engineering challenge.
At this scale, the primary bottleneck is no longer just mathematical correctness—it is hardware memory bandwidth,
interconnect latency, and computational throughput.

I designed and implemented a professional-grade training infrastructure capable of training a 124M parameter GPT-2
model. This project involved orchestrating multi-GPU clusters, maximizing FLOPS through mixed-precision arithmetic, and
managing massive data streams for billions of tokens.

---

## The Problem: The Memory & Compute Wall

A 124M parameter model, while small by modern standards, still requires significant memory for weights, optimizer
states, and gradients. More importantly, stable convergence for LLMs requires massive "effective" batch sizes (e.g.,
0.5M tokens).

A single consumer or professional GPU often lacks the VRAM to hold such a batch. To solve this, the infrastructure must:

1. **Parallelize**: Split the workload across multiple GPUs.
2. **Accumulate**: Simulate large batches through multiple micro-steps.
3. **Optimize**: Use hardware-specific features (Tensor Cores) to accelerate matrix multiplications.

---

## Multi-GPU Orchestration: Distributed Data Parallel (DDP)

I implemented **Distributed Data Parallel (DDP)** using the `nccl` backend. In this architecture, the model is
replicated across $N$ GPUs. Each GPU processes a unique shard of the data, and gradients are synchronized (averaged)
across the cluster through an "All-Reduce" operation after the backward pass.

```python
# DDP Rank & Device Orchestration
ddp_rank = int(os.environ['RANK'])
ddp_local_rank = int(os.environ['LOCAL_RANK'])
ddp_world_size = int(os.environ['WORLD_SIZE'])
device = f'cuda:{ddp_local_rank}'
torch.cuda.set_device(device)

# Initialize process group with NCCL (optimized for NVIDIA)
init_process_group(backend='nccl')
model = DDP(model, device_ids=[ddp_local_rank])
```

---

## Maximizing Throughput: Hardware-Level Optimizations

To push the hardware to its limits, I integrated several modern optimization techniques:

### 1. Mixed Precision (BFloat16 & TF32)

Instead of standard 32-bit floats, I used **BFloat16** for the forward pass. This reduces memory pressure and allows the
GPU to use its **Tensor Cores**, resulting in up to a 4x increase in throughput. I also enabled **TensorFloat-32 (TF32)
** for matrix multiplications on Ampere-class GPUs.

```python
# Enable TF32 for matrix multiplications
torch.set_float32_matmul_precision('high')

# Forward pass with Automatic Mixed Precision
with torch.autocast(device_type='cuda', dtype=torch.bfloat16):
    logits, loss = model(x, y)
```

### 2. Kernel Fusion with `torch.compile`

I utilized `torch.compile` to invoke the **Inductor** compiler. This performs graph-level optimizations and "fuses"
multiple operations (like Add + Tanh + Dropout) into a single GPU kernel. This reduces the "memory trip" overhead,
significantly improving efficiency.

---

## Simulating Supercomputers: Gradient Accumulation

To achieve the 0.5M token batch size required for GPT-2 without running out of memory, I implemented **Gradient
Accumulation**. For each optimizer step, the model performs multiple "micro-steps," summing the gradients over time and
only updating the weights once the target batch size is reached.

```python
# Gradient Accumulation Logic
for micro_step in range(grad_accum_steps):
    # Only synchronize gradients on the final micro-step
    model.require_backward_grad_sync = (micro_step == grad_accum_steps - 1)
    
    with torch.autocast(device_type='cuda', dtype=torch.bfloat16):
        logits, loss = model(x, y)
    
    # Scale loss to normalize the accumulated gradient
    (loss / grad_accum_steps).backward()

# Global gradient clipping to prevent "exploding" updates
torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
optimizer.step()
```

---

## Data Engineering: DataLoaderLite

Training at this scale requires a high-performance data pipeline. I developed `DataLoaderLite`, which:

- Uses **Byte Pair Encoding (BPE)** for efficient tokenization.
- Shards the dataset across GPUs to ensure no two processes see the same data.
- Prefetches data to the GPU to ensure the compute units are never "starved" of work.

---

## Impact

The resulting infrastructure successfully reproduced the training dynamics of the original GPT-2 124M model. By
leveraging DDP and Mixed Precision, I achieved a throughput that allowed for training on billions of tokens in a
fraction of the time required by a naive implementation.

---

## Skills

- **Distributed Systems**: Multi-GPU Synchronisation, Distributed Data Parallel (DDP), torchrun, NCCL Backend, Data Sharding.
- **High-Performance Computing**: Mixed Precision (BF16/TF32), Tensor Cores, Kernel Fusion, torch.compile, JIT Compilation (Inductor).
- **Optimization Engineering**: Gradient Accumulation, Fused AdamW, Gradient Clipping, Cosine Learning Rate Decay.
- **Data Engineering**: Byte Pair Encoding (BPE), Large-Scale Dataset Sharding, FineWeb Processing, Benchmarking (HellaSwag).
