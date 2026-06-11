---

## type: concept domain: Compute & acceleration status: drafted last-reviewed: tags: [compute, tensorrt, inference, quantization, edge]

# Edge Inference


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


## Links

- Related: [[YOLOv8 Detection]], [[Jetson Orin Setup]], [[CUDA & CuPy]], [[OpenCV Pipelines]]
- Parent: [[00 Knowledge Map]]

---

