---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [perception, yolo, object-detection, edge-inference]

# YOLOv8 Detection

> [!question] Explain it cold
> 
> - What does "you only look once" mean architecturally?
> - What's the output of a detection model, and how do you clean overlapping boxes?
> - Why YOLOv8n specifically for edge robotics?

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


## Interview follow-ups

- **Q:** Single-stage vs two-stage detection?
    - **A:** Single-stage (YOLO) predicts all boxes and classes in one pass — fast, good for real-time. Two-stage (Faster R-CNN) proposes regions then classifies them — more accurate, especially on small objects, but slower. Robotics usually takes the speed.
- **Q:** Why do you need NMS?
    - **A:** The detector outputs many overlapping boxes for one object; NMS keeps the highest-confidence box and suppresses overlaps above an IoU threshold, leaving one box per object.
- **Q:** Why the nano model on the robot?
    - **A:** YOLOv8n has the fewest parameters and highest throughput, and the Jetson was simultaneously running ROS2, the LLM, and other nodes — I traded some accuracy for a model that fits the shared compute and hits frame rate.


## Links

- Related: [[OpenCV Pipelines]], [[Edge Inference]], [[CUDA & CuPy]], [[Jetson Orin Setup]], [[LLM Tool Calling]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What makes YOLO "single-stage" and why does it matter for robots? ? One forward pass predicts all boxes + classes simultaneously (vs two-stage region-proposal-then-classify). The single pass is what makes it real-time fast enough for robot perception.

What is NMS and why is it needed? ? Non-Max Suppression: the detector emits many overlapping boxes per object; NMS keeps the highest-confidence one and suppresses overlaps above an IoU threshold.

Why YOLOv8n (nano) on the Jetson? ? Fewest parameters, highest throughput — the Jetson was also running ROS2, Ollama, and other nodes, so it traded accuracy for a model that fits the shared compute budget and hits frame rate.