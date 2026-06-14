---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [real-time, determinism, differentiator]

# Real-Time Determinism

> [!star] Differentiator territory "Real-time" in surgical robotics means _predictable_, not _fast_. This is the conceptual spine connecting iceoryx, control loops, and safety. Own it cold.

> [!question] Explain it cold
> 
> - What does "real-time" actually mean? (Hint: not "fast.")
> - Hard vs soft real-time — example of each.
> - What is jitter, and why is it worse than latency for a control loop?

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

## Where I've used it

- **Articulus**: the entire control stack was built around bounded, predictable timing — that's _why_ iceoryx zero-copy mattered ([[iceoryx Zero-Copy]]): constant latency independent of payload removes a jitter source. No-alloc hot paths, deterministic dispatch via mutually-exclusive callback groups ([[ROS2 Executors]]).
- **WALL-E V3**: the 200 Hz gimbal loop on the Arduino UNO is a hard-real-time inner loop — a complementary filter + LQG at fixed dt. Jitter there shows up as attitude estimate drift.

## Interview follow-ups

- **Q:** Is a 1 GHz CPU more real-time than a 100 MHz MCU?
    - **A:** Not necessarily. Real-time is about bounded, predictable response, not raw speed. A simple MCU with no OS jitter can be _more_ deterministic than a fast CPU running a general-purpose OS.
- **Q:** Why is dynamic allocation banned on real-time paths?
    - **A:** malloc can take unbounded time (fragmentation, lock contention, syscalls) and can fail — non-deterministic. Pre-allocate everything up front.
- **Q:** What's priority inversion and how do you prevent it?
    - **A:** A high-priority task blocked by a low-priority task holding a shared lock. Prevent with priority inheritance or priority-ceiling protocols on the mutex.

## Profiling & Measuring Real-Time Performance

### `cyclictest` — measure wakeup latency jitter

Sets a high-resolution timer to fire every X microseconds and records how late the actual wakeup is each iteration. Gives you a histogram of jitter across thousands of samples.

```bash
cyclictest -l100000 -m -n -a0 -t1 -p99 -i400 -h400
# -l: iterations  -m: mlockall  -p99: RT priority  -i: interval (μs)  -h: histogram buckets
```

What good vs bad looks like:
- **Non-RT kernel**: max jitter 500μs–2ms (unacceptable for 1kHz control)
- **`PREEMPT_RT` kernel, tuned**: max jitter < 100μs, ideally < 50μs

### `perf` — find *why* jitter is happening

Profiles the CPU while `cyclictest` runs to reveal what's eating your RT budget:

```bash
perf record cyclictest -l100000 -m -n -a0 -t1 -p99 -i400
perf report   # shows hottest functions / kernel paths
```

Common culprits `perf` reveals:
- Kernel interrupt handlers preempting the RT task
- Page faults (forgot `mlockall` → memory not locked in RAM)
- Scheduler contention between RT and non-RT threads
- Cache misses in the hot path

### The workflow together

1. Run `cyclictest` → quantify how bad your jitter is
2. Run `perf record cyclictest` → identify the source
3. Fix (pin IRQs, `mlockall`, isolate CPUs with `isolcpus`, tune scheduler)
4. Re-run `cyclictest` → verify improvement

### Why this matters for RSE interviews

A 1kHz control loop has a 1ms deadline. If the OS wakes your thread 600μs late, half the deadline is gone before you do any work. Being able to say "I use `cyclictest` to measure jitter and `perf` to find the source" is differentiator-level — most candidates don't know this exists.

### Tools summary for "what tools do you use?"

| Layer | Tools |
|---|---|
| Profiling & observability | `perf`, `htop`/`btop`, `ros2 topic hz`, `ros2 topic delay` |
| RT latency measurement | `cyclictest`, `perf record` |
| Build & toolchain | `colcon`, `ament_cmake`, `cmake`, `gdb` |
| Simulation | Isaac Lab, Gazebo, `rosbag2` |
| Embedded / RT | FreeRTOS, micro-ROS, `PREEMPT_RT` kernel |

One-liner for interviews:
> "I use `perf` and `htop` for system-level profiling, `cyclictest` to measure RT jitter, `colcon`/`cmake` for builds, `gdb` for debugging C++ nodes, and Isaac Lab or Gazebo for simulation. On the embedded side I work with FreeRTOS and micro-ROS."

---

## Gotchas / what trips me up

- Saying "real-time = fast." It's _predictable_. This is the single most common misconception and a great place to sound senior by correcting it.
- Budgeting for average instead of worst-case execution time.

## Links

- Related: [[iceoryx Zero-Copy]], [[ROS2 Executors]], [[Watchdog Timers]], [[Safety-Critical Architecture]], [[Latency Budgets]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does "real-time" actually guarantee? ? A bounded, predictable response within a deadline every time — determinism, not raw speed.

Latency vs jitter — which hurts a fixed-dt control loop more, and why? ? Jitter (variance in latency). The loop's discretization assumes a fixed dt; wandering dt corrupts the control math even if average latency is fine.

Why is malloc banned on real-time hot paths? ? It can take unbounded time and may fail (fragmentation, lock contention) — non-deterministic. Pre-allocate instead.

What is priority inversion and the fix? ? A high-priority task is blocked by a low-priority task holding a shared lock; fix with priority inheritance or priority-ceiling protocols.

What does `cyclictest` measure and what are good/bad numbers? ? Wakeup latency jitter — how late the OS wakes a timer-driven RT task. Bad: 500μs–2ms on stock kernel. Good: <100μs on PREEMPT_RT tuned system.

How do you use `perf` and `cyclictest` together? ? `perf record cyclictest ...` then `perf report` — cyclictest quantifies the jitter, perf reveals which kernel path or function is causing it (IRQ handlers, page faults, scheduler contention).

Name the common causes of RT jitter that `perf` reveals. ? Kernel interrupt handlers, page faults (missing mlockall), scheduler contention between RT and non-RT threads, cache misses on the hot path.