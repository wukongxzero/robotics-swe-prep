---

## type: concept domain: Compute & acceleration status: drafted last-reviewed: tags: [compute, jetson, edge, embedded]

# Jetson Orin Setup

> [!question] Explain it cold
> 
> - What is a Jetson and why use one for robotics?
> - What's JetPack and how do you flash a Jetson?
> - What's the unified-memory advantage for perception?

---

## Core idea

The NVIDIA Jetson is an embedded GPU compute module — a real CUDA GPU + ARM CPU in a low-power edge form factor. It's the standard "robot brain" when you need on-device deep-learning inference (perception, LLMs) without a desktop GPU or cloud round-trip. The Orin Nano is the small/cheap entry point.

## Key facts & formulas

- **Architecture**: ARM CPU + Ampere CUDA GPU sharing **unified memory** — CPU and GPU access the same physical RAM, so you avoid host↔device copies that dominate on discrete GPUs. Big win for perception pipelines moving images around.
- **JetPack**: NVIDIA's SDK bundle — L4T (Linux for Tegra, Ubuntu-based), CUDA, cuDNN, TensorRT, OpenCV, multimedia. JetPack version pins your CUDA/TensorRT versions (compatibility headaches live here).
- **SDK Manager**: the host-side tool to flash JetPack onto the Jetson over USB (recovery mode). The Orin Nano can also boot JetPack from SD/NVMe.
- **Power modes**: configurable power/clock budgets (nvpmodel) — trade watts for TOPS. Matters when the board runs perception + control + LLM simultaneously and thermals/power are constrained.


## Interview follow-ups

- **Q:** Why a Jetson over a Raspberry Pi or a laptop?
    - **A:** It has a real CUDA GPU in an embedded power envelope — on-device deep-learning inference (detection, LLM) at the edge, no cloud latency, with unified CPU/GPU memory. A Pi can't run the models; a laptop isn't an embeddable robot brain.
- **Q:** What's the unified-memory advantage?
    - **A:** CPU and GPU share the same physical RAM, so you skip the host↔device copies that bottleneck discrete GPUs — directly faster for image-heavy perception.
- **Q:** What bites you in Jetson setup?
    - **A:** JetPack version pinning — CUDA/TensorRT/cuDNN versions are coupled to the JetPack release, so library compatibility (and getting frameworks to match) is the recurring pain.


## Links

- Related: [[CUDA & CuPy]], [[Docker for ROS]], [[NixOS for Jetson]], [[Edge Inference]], [[YOLOv8 Detection]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Why use a Jetson for robotics? ? Real CUDA GPU + ARM CPU in a low-power embedded form factor — on-device deep-learning inference (perception, LLMs) at the edge without cloud latency, with unified CPU/GPU memory.

What is JetPack and why does its version matter? ? NVIDIA's SDK bundle (L4T Ubuntu, CUDA, cuDNN, TensorRT, OpenCV). The JetPack version pins coupled CUDA/TensorRT versions — mismatches are the classic setup failure.

Unified memory advantage on Jetson? ? CPU and GPU share the same physical RAM, avoiding the host↔device copies that bottleneck discrete GPUs — faster image-heavy perception.