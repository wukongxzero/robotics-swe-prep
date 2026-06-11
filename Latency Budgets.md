---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, latency, real-time, articulus, differentiator, hands-on]

# Latency Budgets

> [!star] Hands-on — the thread that ties your whole stack together Latency budgeting is _why_ every low-level choice (zero-copy, CAN, deterministic dispatch) was made. This note is the narrative that connects the embedded/middleware/control notes into one engineering story. Provenance: hands-on.


---

## Core idea

A latency budget is the total allowable delay from cause to effect, divided up across every stage of the pipeline. In surgical teleop the surgeon is in a closed loop with the instrument (and haptics), so the _whole_ loop must close within a tight, bounded time — and every component gets a slice of that budget. Exceed it and the system feels laggy/unstable, which is a safety problem.

## Key facts & formulas

- **The loop**: surgeon hand motion → master sensing → [[Teleoperation & Motion Scaling|filtering/scaling]] → comms → [[Inverse Kinematics & DLS|IK]]/control → [[Motor Control|actuation]] → instrument moves → force sensed → [[Force Feedback & Haptics|haptics]] back to hand. Every arrow has a latency, and they _sum_.
- **Why bounded matters more than small**: it's not just average latency — it's **worst-case** and **jitter** ([[Real-Time Determinism]]). A loop that's usually 2 ms but occasionally 20 ms is worse than a steady 5 ms, because the surgeon's loop and any [[Force Feedback & Haptics|haptic]] stability margin assume predictability.
- **Where latency is spent**: serialization/copies, network/bus transit, queueing (deep buffers!), computation (IK, filtering), actuation, sensor sampling.
- **Where it's saved (the choices I made)**:
    - [[iceoryx Zero-Copy|Zero-copy]] — removes serialize/copy/deserialize; constant regardless of payload. Saves the comms slice.
    - [[CAN Bus|CAN priority arbitration]] — bounded latency for high-priority motor messages.
    - [[ROS2 QoS|depth-1 QoS]] — no stale-data queue buildup.
    - [[ROS2 Executors|deterministic dispatch]] — safety/control path never preempted.
    - No dynamic allocation on the hot path — no unbounded pauses.
- **The unifying story**: each of those isn't a separate "cool tech" — they're all line items spending or protecting the latency budget that the surgical loop demands.


## Links

- Related: [[iceoryx Zero-Copy]], [[CAN Bus]], [[Real-Time Determinism]], [[ROS2 QoS]], [[ROS2 Executors]], [[Safety-Critical Architecture]], [[Teleoperation & Motion Scaling]], [[Force Feedback & Haptics]]
- Parent: [[00 Knowledge Map]]

---

