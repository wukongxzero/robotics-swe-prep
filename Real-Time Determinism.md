---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [real-time, determinism, differentiator]

# Real-Time Determinism

> [!star] Differentiator territory "Real-time" in surgical robotics means _predictable_, not _fast_. This is the conceptual spine connecting iceoryx, control loops, and safety. Own it cold.


---

## Core idea

Real-time means **deterministic timing** — the system guarantees a response within a bounded deadline, _every_ time. A slow-but-predictable system can be real-time; a fast-on-average-but-occasionally-stalling system is not. The enemy is variance, not magnitude.

## Key facts & formulas

- **Hard real-time**: missing a deadline is a system failure (motor control loop, safety monitor). **Soft real-time**: missing degrades quality but isn't catastrophic (video stream, UI).
- **Latency** = delay from cause to response. **Jitter** = variance in that latency. A control loop tuned for a fixed dt breaks when dt wanders — jitter corrupts the discretization, not just the speed.
- **WCET** (Worst-Case Execution Time): real-time analysis budgets the _worst_ case, not the average. You size deadlines against WCET.
- **Sources of non-determinism**: dynamic memory allocation (heap can block/fragment), garbage collection, page faults, priority inversion, unbounded queues, cache misses, lock contention.
- **Real-time-safe coding**: pre-allocate (no malloc on the hot path), bounded loops, lock-free or priority-inheritance mutexes, pin threads, `RT_PREEMPT` kernel for Linux RT.
- **Priority inversion**: a low-priority task holding a lock blocks a high-priority task. Fix: priority inheritance / ceiling protocols. (Mars Pathfinder is the classic war story.)


## Links

- Related: [[iceoryx Zero-Copy]], [[ROS2 Executors]], [[Watchdog Timers]], [[Safety-Critical Architecture]], [[Latency Budgets]]
- Parent: [[00 Knowledge Map]]

---

