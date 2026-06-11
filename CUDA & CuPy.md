---

## type: concept domain: Compute & acceleration status: drafted last-reviewed: tags: [compute, cuda, gpu, cupy, kapila-final]

# CUDA & CuPy

> [!question] Explain it cold
> 
> - What's the CUDA execution model (threads/blocks/grids)?
> - When does GPU offload actually pay off, and when does it not?
> - What does CuPy give you over writing raw CUDA C?

---

## Core idea

CUDA is NVIDIA's model for general-purpose GPU compute: launch the _same_ function (kernel) across thousands of threads operating on different data — massively data-parallel. It pays off when you have lots of independent identical work (image ops, matrix math, inference); it's a loss when work is small, serial, or copy-dominated.

## Key facts & formulas

- **Execution hierarchy**: **thread** → **block** (threads sharing fast shared memory, synchronizable) → **grid** (all blocks for a kernel launch). You pick block/grid dimensions to cover your data. Threads in a **warp** (32) execute in lockstep — divergent branches serialize (warp divergence).
- **SIMT** (Single Instruction Multiple Thread): one instruction stream, many threads, different data.
- **Memory hierarchy**: registers (per-thread) > shared (per-block, fast) > global (large, slow) > host. Performance is usually about memory access patterns (coalescing) more than raw FLOPs.
- **The offload tax**: host↔device copies. On a discrete GPU this often dominates; on the [[Jetson Orin Setup|Jetson's unified memory]] it's cheaper. Offload only when compute >> transfer cost.
- **CuPy**: NumPy-compatible array library backed by CUDA — write `cp.array` ops, run on GPU, near-zero code change from NumPy. **RawKernel** lets you drop to a hand-written CUDA C kernel when you need it. The pragmatic middle: NumPy ergonomics, CUDA speed, escape hatch for custom kernels.

## Where I've used it

- **Kapila final**: Jetson/CUDA material — OpenCV GPU pipelines, **CuPy RawKernel**, CNN inference via jetson_inference. Register-level embedded plus GPU compute were the two poles of the course.
- **WALL-E V3**: the GPU on the [[Jetson Orin Setup|Orin Nano]] ran perception ([[YOLOv8 Detection]]) and the LLM — the practical "why CUDA matters on a robot" context.

## Interview follow-ups

- **Q:** CUDA thread hierarchy?
    - **A:** Threads grouped into blocks (shared memory, can sync), blocks into a grid per kernel launch. Warps of 32 threads run in lockstep — divergent branches within a warp serialize.
- **Q:** When does GPU offload not help?
    - **A:** Small or serial workloads, or when host↔device copy cost exceeds the compute saved. GPUs win on large, independent, identical work; the transfer tax can erase gains otherwise (less so on Jetson unified memory).
- **Q:** Why CuPy instead of raw CUDA?
    - **A:** NumPy-compatible API means GPU acceleration with almost no code change, and RawKernel is the escape hatch for a custom CUDA C kernel when the vectorized ops aren't enough. Productivity without giving up control.

## Gotchas / what trips me up

- Assuming GPU is always faster — copy overhead and small problems kill it.
- Warp divergence from branchy kernels silently tanking throughput.
- Memory-access patterns (coalescing) matter more than FLOP count.

## Links

- Related: [[Jetson Orin Setup]], [[Edge Inference]], [[YOLOv8 Detection]], [[OpenCV Pipelines]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

CUDA execution hierarchy? ? Threads → blocks (shared memory, synchronizable) → grid (per kernel launch). Warps of 32 threads run in lockstep; divergent branches within a warp serialize.

When does GPU offload NOT pay off? ? Small or serial workloads, or when host↔device copy cost exceeds compute saved. GPUs win on large, independent, identical work (copy tax is cheaper on Jetson unified memory).

What does CuPy give you, and what's RawKernel for? ? NumPy-compatible GPU arrays — acceleration with near-zero code change. RawKernel is the escape hatch to write a custom CUDA C kernel when vectorized ops aren't enough.