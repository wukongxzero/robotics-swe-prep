---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [ml, llm, tool-calling, wall-e]

# LLM Tool Calling

> [!question] Explain it cold
> 
> - What is LLM tool/function calling and how does it bridge language to action?
> - How do you run an LLM on an edge robot, and what's the catch?
> - How do you keep an LLM's output safe to feed a real actuator?

---

## Core idea

LLM tool calling = the model, instead of replying in prose, emits a **structured call** (a function name + typed arguments, usually JSON) that maps to a real capability. For a robot, this turns "go pick up the red block" into a structured command the control stack can execute — the LLM is a natural-language _front-end_ to a constrained set of robot actions, not the controller itself.

## Key facts & formulas

- **Mechanism**: you define a schema of available tools/functions (name, parameters, types); the LLM parses the user's intent and outputs which tool to call with which args. The runtime validates and dispatches it. The model never touches the actuator — it selects from a vetted action set.
- **Why structured output matters**: free-text is unsafe and unparseable for control. Constraining the model to a fixed schema (function calling, or grammar/JSON-mode decoding) makes the output a finite, validatable command space.
- **Edge LLM**: running a small model locally (Ollama + a quantized model) avoids cloud latency/connectivity dependence — but small models are weaker at parsing and you pay compute on a device already running perception + control.
- **Safety layering**: the LLM output is a _suggestion_ that passes through validation, bounds-checking, and the [[ROS2 Node & Lifecycle|state machine]] before any motion. Never wire model output straight to actuators — the deterministic safety layer sits between.

## Where I've used it

- **WALL-E V3**: **LLaMA 3.2:3b via Ollama** for natural-language command parsing — spoken/typed commands → structured robot commands routed through the `controller_node` and MANUAL/AUTONOMOUS/IDLE [[ROS2 Node & Lifecycle|state machine]]. Ran in a Docker Compose service (`ollama/ollama`) alongside the ROS2 stack on the Jetson Orin Nano. The 3b model was the size-vs-capability tradeoff for on-device parsing while [[YOLOv8 Detection|YOLOv8n]] and the control nodes shared the GPU.
- The LLM is the _intent_ layer; the deterministic stack underneath is what's actually safe — an important framing for a safety-critical-minded interviewer.

## Interview follow-ups

- **Q:** How does an LLM control a robot safely?
    - **A:** It doesn't directly. It does natural-language → structured tool call against a vetted action schema; the runtime validates the call, bounds-checks args, and routes through the state machine. The LLM proposes intent; a deterministic layer executes only safe, validated commands.
- **Q:** Why run the model on-device instead of the cloud?
    - **A:** Latency and connectivity independence — a robot can't depend on a network round-trip for commands. The cost is a smaller, weaker model and competing for the Jetson's compute, so I used a 3b model and kept the LLM off the real-time path.
- **Q:** How do you make LLM output parseable for control?
    - **A:** Constrain it — function-calling schemas or JSON/grammar-constrained decoding — so the output is a finite, typed, validatable command set rather than free text.

## Gotchas / what trips me up

- Treating the LLM as the controller — it's the intent front-end; the deterministic stack is the controller.
- Forgetting on-device small models are meaningfully weaker at structured parsing.
- Not validating/bounding model output before it reaches motion.

## Links

- Related: [[ROS2 Node & Lifecycle]], [[YOLOv8 Detection]], [[Docker for ROS]], [[Jetson Orin Setup]], [[Safety-Critical Architecture]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is LLM tool calling and how does it bridge language to robot action? ? The model emits a structured function call (name + typed args, JSON) from a vetted tool schema instead of prose. It's a natural-language front-end that maps intent to a constrained, validatable action set — not the controller itself.

How do you keep LLM-driven robot commands safe? ? The LLM only proposes a structured call; a deterministic layer validates, bounds-checks, and routes it through the state machine before any motion. Model output never wires straight to actuators.

Why run the LLM on-device (Ollama, 3b model) and what's the catch? ? Latency and connectivity independence — no network round-trip for commands. The catch: smaller/weaker model and competition for the Jetson's compute, so keep it off the real-time path.