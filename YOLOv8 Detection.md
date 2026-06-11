---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [perception, yolo, object-detection, edge-inference]

# YOLOv8 Detection


---

## Core idea

YOLO is a **single-stage** object detector: one forward pass over the whole image predicts all bounding boxes + classes simultaneously, rather than proposing regions then classifying them (two-stage, like Faster R-CNN). That single-pass design is what makes it fast enough for real-time robot perception. YOLOv8 is the Ultralytics iteration; the **n** (nano) variant is the smallest, for edge devices.

## Key facts & formulas

- **Single-stage vs two-stage**: single-stage (YOLO, SSD) = one network, fast, slightly less accurate on small objects. Two-stage (Faster R-CNN) = region proposal then classify, more accurate, slower. Robots usually need the speed.
- **Output per detection**: bounding box (x, y, w, h), objectness/confidence, class probabilities. YOLOv8 is anchor-free (predicts box centers directly) and decoupled head (separate classification/regression branches) vs older anchor-based YOLOs.
- **NMS (Non-Max Suppression)**: the model emits many overlapping boxes; NMS keeps the highest-confidence box and suppresses others with IoU above a threshold. Essential post-processing.
- **IoU** (Intersection over Union): overlap metric for matching/NMS/evaluation; mAP is the standard accuracy metric.
- **Model sizes** n/s/m/l/x trade accuracy for speed/size. **YOLOv8n** = nano, fewest params, fastest — the right pick for a Jetson where you're sharing compute with everything else.
- **Edge deployment**: export to TensorRT/ONNX for the Jetson, quantize (FP16/INT8) for throughput — see [[Edge Inference]].


## Links

- Related: [[OpenCV Pipelines]], [[Edge Inference]], [[CUDA & CuPy]], [[Jetson Orin Setup]], [[LLM Tool Calling]]
- Parent: [[00 Knowledge Map]]

---

