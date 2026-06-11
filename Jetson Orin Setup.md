---

## type: concept domain: Compute & acceleration status: drafted last-reviewed: tags: [compute, jetson, edge, embedded]

# Jetson Orin Setup


---

## Core idea

The NVIDIA Jetson is an embedded GPU compute module — a real CUDA GPU + ARM CPU in a low-power edge form factor. It's the standard "robot brain" when you need on-device deep-learning inference (perception, LLMs) without a desktop GPU or cloud round-trip. The Orin Nano is the small/cheap entry point.

## Key facts & formulas

- **Architecture**: ARM CPU + Ampere CUDA GPU sharing **unified memory** — CPU and GPU access the same physical RAM, so you avoid host↔device copies that dominate on discrete GPUs. Big win for perception pipelines moving images around.
- **JetPack**: NVIDIA's SDK bundle — L4T (Linux for Tegra, Ubuntu-based), CUDA, cuDNN, TensorRT, OpenCV, multimedia. JetPack version pins your CUDA/TensorRT versions (compatibility headaches live here).
- **SDK Manager**: the host-side tool to flash JetPack onto the Jetson over USB (recovery mode). The Orin Nano can also boot JetPack from SD/NVMe.
- **Power modes**: configurable power/clock budgets (nvpmodel) — trade watts for TOPS. Matters when the board runs perception + control + LLM simultaneously and thermals/power are constrained.


## Links

- Related: [[CUDA & CuPy]], [[Docker for ROS]], [[NixOS for Jetson]], [[Edge Inference]], [[YOLOv8 Detection]]
- Parent: [[00 Knowledge Map]]

---

