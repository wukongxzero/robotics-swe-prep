---
type: concept
domain: ROS2 & middleware
status: drafted
last-reviewed: 2026-06-02
tags: [ros2, concurrency, real-time]
---

# ROS2 Executors

> [!question] Explain it cold
> *Answer from memory first.*
>
> - What is it, in two sentences?
> - When/why would you reach for it?
> - What's the one thing that trips people up?

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

## Where I've used it
- **Articulus** real-time stack: deterministic dispatch mattered for the control loop — mutually-exclusive grouping on the safety-critical path so a slow service call could never preempt the loop.
- **WALL-E V3**: `controller_node` + `state_machine` (MANUAL/AUTONOMOUS/IDLE) running alongside the `uno_bridge` and `mega_node` — sensor callbacks must not be starved by the LLaMA command-parse path.
- **FENCE-BOT**: `robot_controller` at 50 Hz consuming `/vr_pose`; the publish cadence has to hold regardless of logging callbacks.

## Interview follow-ups
- **Q:** A timer is set to 10 ms but its callback takes 15 ms on a single-threaded executor. What happens?
  - **A:** The next timer fire is delayed — the executor can't dispatch it until the current callback returns. You get drift/missed deadlines, not concurrency. Fix: shorten the callback, move work to a reentrant group on a multi-threaded executor, or offload.
- **Q:** A slow service callback is starving your sensor subscription. How do you fix it without rewriting logic?
  - **A:** Put the service and the subscription in *separate* callback groups and use a MultiThreadedExecutor so they can run on different threads. Or keep the sensor path mutually-exclusive and isolated.
- **Q:** What are the thread-safety implications of a reentrant group?
  - **A:** Two instances of the same callback (or two callbacks in the group) can run simultaneously — shared state needs locking, and you must avoid non-reentrant resources. Concurrency you asked for is concurrency you have to make safe.

## Gotchas / what trips me up
- Saying "MultiThreadedExecutor makes callbacks parallel" — *only* if their callback groups allow it. Default groups are mutually exclusive, so a naive switch changes nothing.
- Conflating the executor (threading) with QoS (delivery guarantees). Different layers. See [[ROS2 QoS]].

## Links
- Related: [[ROS2 QoS]], [[ROS2 Node & Lifecycle]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does a ROS2 executor own that a node does not?
?
The spin loop — it collects ready callbacks into a wait set and dispatches them, deciding when and on how many threads they run.

Default callback group type, and what it guarantees?
?
MutuallyExclusive — callbacks in the same group never overlap, even on a MultiThreadedExecutor.

On a single-threaded executor, what happens when a callback runs longer than a timer's period?
?
The next timer fire is delayed until the current callback returns — drift and missed deadlines, no concurrency.
