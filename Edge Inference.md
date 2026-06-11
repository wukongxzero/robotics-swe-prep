---

## type: concept domain: Compute & acceleration status: drafted last-reviewed: tags: [compute, tensorrt, inference, quantization, edge]

# Edge Inference

> [!question] Explain it cold
> 
> - What does TensorRT do to a trained model, and why?
> - What is quantization and what does it trade?
> - Why not just run the PyTorch model on the robot?

---

## Core idea

Edge inference = running trained neural nets _on the robot_ under tight compute/power/latency budgets. You don't deploy the training framework — you compile and optimize the model for the target hardware. On NVIDIA edge, that's **TensorRT**: it takes a trained model and produces a hardware-optimized inference engine.

## Key facts & formulas

- **TensorRT optimizations**: layer/tensor **fusion** (combine ops to cut memory traffic), **precision calibration** (FP32→FP16/INT8), kernel auto-tuning for the specific GPU, and memory planning. Output is a serialized engine specific to that hardware/precision.
- **Quantization**: represent weights/activations in lower precision.
    - **FP16**: ~2× throughput/memory, minimal accuracy loss — usually free on Jetson.
    - **INT8**: bigger speedup, needs **calibration** (a representative dataset to set scales) and risks more accuracy loss.
    - Trade: speed/memory vs accuracy. Quantize as far as accuracy tolerates.
- **The deployment path**: train (PyTorch) → export (ONNX) → build TensorRT engine → run on device. Don't ship the training stack.
- **jetson_inference**: NVIDIA's ready-made library/wrappers for running classification/detection/segmentation on Jetson with TensorRT under the hood — fast path to deployed CNNs.
- **Why not raw PyTorch on-device**: works, but leaves large speed/memory on the table — no fusion, no INT8, framework overhead. On a shared-compute robot that's the difference between hitting frame rate and not.

## Where I've used it

- **Kapila final**: CNN inference via **jetson_inference** on the Jetson was explicit course material (with CuPy RawKernel, OpenCV pipelines).
- **WALL-E V3**: [[YOLOv8 Detection|YOLOv8n]] on the [[Jetson Orin Setup|Orin Nano]] — the case for exporting to TensorRT rather than running PyTorch directly, since the board shared compute with ROS2 and the LLM.

## Interview follow-ups

- **Q:** What does TensorRT actually do?
    - **A:** Compiles a trained model into a hardware-specific inference engine — fuses layers, calibrates precision (FP16/INT8), auto-tunes kernels, plans memory. The result is much faster inference than running the training framework on-device.
- **Q:** FP16 vs INT8 quantization?
    - **A:** FP16 roughly doubles throughput with minimal accuracy loss — usually a free win. INT8 is faster still but needs calibration data to set quantization scales and risks more accuracy loss. Pick based on accuracy headroom.
- **Q:** Why not run PyTorch directly on the Jetson?
    - **A:** It works but wastes performance — no operator fusion, no INT8, framework overhead. On a robot sharing compute across perception, control, and an LLM, the TensorRT engine is what makes frame rate achievable.

## Gotchas / what trips me up

- INT8 without proper calibration → accuracy cliff.
- TensorRT engines are hardware/precision-specific — not portable across Jetson models.
- Forgetting the ONNX export step in the path.

## Links

- Related: [[YOLOv8 Detection]], [[Jetson Orin Setup]], [[CUDA & CuPy]], [[OpenCV Pipelines]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does TensorRT do to a trained model? ? Compiles it into a hardware-specific inference engine: layer/tensor fusion, precision calibration (FP16/INT8), kernel auto-tuning, memory planning — far faster than running the training framework on-device.

FP16 vs INT8 quantization tradeoff? ? FP16: ~2× throughput, minimal accuracy loss — usually free. INT8: faster still but needs calibration data and risks more accuracy loss. Quantize as far as accuracy tolerates.

Why not run PyTorch directly on the robot? ? It works but wastes performance — no fusion, no INT8, framework overhead. On shared-compute robots the TensorRT engine is what makes frame rate achievable.