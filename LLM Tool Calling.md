---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [ml, llm, tool-calling, wall-e]

# LLM Tool Calling


---

## Core idea

LLM tool calling = the model, instead of replying in prose, emits a **structured call** (a function name + typed arguments, usually JSON) that maps to a real capability. For a robot, this turns "go pick up the red block" into a structured command the control stack can execute — the LLM is a natural-language _front-end_ to a constrained set of robot actions, not the controller itself.

## Key facts & formulas

- **Mechanism**: you define a schema of available tools/functions (name, parameters, types); the LLM parses the user's intent and outputs which tool to call with which args. The runtime validates and dispatches it. The model never touches the actuator — it selects from a vetted action set.
- **Why structured output matters**: free-text is unsafe and unparseable for control. Constraining the model to a fixed schema (function calling, or grammar/JSON-mode decoding) makes the output a finite, validatable command space.
- **Edge LLM**: running a small model locally (Ollama + a quantized model) avoids cloud latency/connectivity dependence — but small models are weaker at parsing and you pay compute on a device already running perception + control.
- **Safety layering**: the LLM output is a _suggestion_ that passes through validation, bounds-checking, and the [[ROS2 Node & Lifecycle|state machine]] before any motion. Never wire model output straight to actuators — the deterministic safety layer sits between.


## Links

- Related: [[ROS2 Node & Lifecycle]], [[YOLOv8 Detection]], [[Docker for ROS]], [[Jetson Orin Setup]], [[Safety-Critical Architecture]]
- Parent: [[00 Knowledge Map]]

---

