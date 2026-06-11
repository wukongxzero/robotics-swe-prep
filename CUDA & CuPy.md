---

## type: concept domain: Compute & acceleration status: drafted last-reviewed: tags: [compute, cuda, gpu, cupy, kapila-final]

# CUDA & CuPy


---

## Core idea

CUDA is NVIDIA's model for general-purpose GPU compute: launch the _same_ function (kernel) across thousands of threads operating on different data — massively data-parallel. It pays off when you have lots of independent identical work (image ops, matrix math, inference); it's a loss when work is small, serial, or copy-dominated.

## Key facts & formulas

- **Execution hierarchy**: **thread** → **block** (threads sharing fast shared memory, synchronizable) → **grid** (all blocks for a kernel launch). You pick block/grid dimensions to cover your data. Threads in a **warp** (32) execute in lockstep — divergent branches serialize (warp divergence).
- **SIMT** (Single Instruction Multiple Thread): one instruction stream, many threads, different data.
- **Memory hierarchy**: registers (per-thread) > shared (per-block, fast) > global (large, slow) > host. Performance is usually about memory access patterns (coalescing) more than raw FLOPs.
- **The offload tax**: host↔device copies. On a discrete GPU this often dominates; on the [[Jetson Orin Setup|Jetson's unified memory]] it's cheaper. Offload only when compute >> transfer cost.
- **CuPy**: NumPy-compatible array library backed by CUDA — write `cp.array` ops, run on GPU, near-zero code change from NumPy. **RawKernel** lets you drop to a hand-written CUDA C kernel when you need it. The pragmatic middle: NumPy ergonomics, CUDA speed, escape hatch for custom kernels.


## Links

- Related: [[Jetson Orin Setup]], [[Edge Inference]], [[YOLOv8 Detection]], [[OpenCV Pipelines]]
- Parent: [[00 Knowledge Map]]

---

