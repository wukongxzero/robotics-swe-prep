---
type: concept
domain: ROS2 & middleware
status: drafted
last-reviewed: 2026-06-02
tags: [ros2, concurrency, real-time]
---

# ROS2 Executors


---

## Core idea
An executor is the thing that actually *runs* your callbacks — it owns the spin loop, collects ready work (subscriptions, timers, services, clients) into a wait set, and dispatches it. The node defines *what* the callbacks are; the executor decides *when and on how many threads* they run.

## Key facts & formulas
- **SingleThreadedExecutor** — one thread, callbacks run one at a time in the order they became ready. Simple and deterministic, but one slow callback blocks everything else.
- **MultiThreadedExecutor** — a thread pool; callbacks *can* run concurrently, subject to callback-group rules.
- **Callback groups** decide concurrency, not the executor alone:
  - **MutuallyExclusive** (default) — callbacks in the same group never overlap, even on a multi-threaded executor.
  - **Reentrant** — callbacks in the group may run concurrently / re-enter.
- **Spin variants**: `spin()` blocks forever; `spin_once()` handles one ready item; `spin_some()` / `spin_all()` drain currently-ready work without waiting.
- A callback's group assignment is the real concurrency control surface — change behavior by regrouping callbacks, not by swapping executors blindly.


## Links
- Related: [[ROS2 QoS]], [[ROS2 Node & Lifecycle]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

