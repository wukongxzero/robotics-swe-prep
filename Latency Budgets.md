---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, latency, real-time, articulus, differentiator, hands-on]

# Latency Budgets

> [!star] Hands-on — the thread that ties your whole stack together Latency budgeting is _why_ every low-level choice (zero-copy, CAN, deterministic dispatch) was made. This note is the narrative that connects the embedded/middleware/control notes into one engineering story. Provenance: hands-on.

> [!question] Explain it cold
> 
> - What is an end-to-end latency budget and why does surgery impose one?
> - Walk the latency path from surgeon's hand to instrument motion to haptic feedback.
> - Which engineering choices "spend" or "save" latency?

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


## Interview follow-ups

- **Q:** What's an end-to-end latency budget?
    - **A:** The total allowable delay around the control loop, allocated across every stage — sensing, comms, computation, actuation, feedback. In surgical teleop the surgeon is in the loop, so the whole thing must close within a tight, bounded, predictable time; each component gets a slice.
- **Q:** Why is jitter as important as latency?
    - **A:** The surgeon's feedback loop and haptic stability assume predictable timing. A loop that occasionally spikes is worse than a steady slightly-higher latency — variance breaks the assumptions stability rests on. Worst-case and jitter, not just average.
- **Q:** Connect your low-level choices to the latency budget.
    - **A:** Zero-copy removed the serialize/copy comms cost; CAN priority arbitration bounded motor-message latency; depth-1 QoS avoided stale queue buildup; deterministic executor dispatch kept the safety path un-preempted; no hot-path allocation avoided unbounded pauses. Every one was spending or protecting the budget the surgical loop demanded.


## Links

- Related: [[iceoryx Zero-Copy]], [[CAN Bus]], [[Real-Time Determinism]], [[ROS2 QoS]], [[ROS2 Executors]], [[Safety-Critical Architecture]], [[Teleoperation & Motion Scaling]], [[Force Feedback & Haptics]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is an end-to-end latency budget and why does surgical teleop impose one? ? The total allowable loop delay, allocated across each stage (sensing, comms, compute, actuation, feedback). The surgeon is in a closed loop with the instrument/haptics, so the whole loop must close within a tight, bounded, predictable time.

Why is jitter as important as average latency? ? The surgeon's feedback loop and haptic stability assume predictable timing; an occasional spike is worse than steady slightly-higher latency. Bound the worst case and jitter, not just the average.

Connect zero-copy, CAN, QoS depth-1, and deterministic dispatch to one idea. ? They're all line items spending or protecting the surgical loop's latency budget: zero-copy saves comms cost, CAN bounds motor-message latency, depth-1 avoids stale queues, deterministic dispatch protects the safety path.