---
type: concept
domain: Real-time & embedded
status: drafted
last-reviewed: 2026-06-02
tags: [rtos, scheduling, embedded, real-time, priority, preemption]
---

# RTOS Fundamentals

> [!question] Explain it cold
> - What makes an OS "real-time" vs a general-purpose OS?
> - What is preemptive scheduling and why does it matter?
> - What is priority inversion and how do you fix it?

---

## Core idea

An RTOS guarantees that a task completes within a bounded time (deadline). A general-purpose OS (Linux, Windows) optimizes for throughput and fairness — it may delay your task for milliseconds. An RTOS sacrifices throughput for **determinism**: you can prove worst-case response time.

---

## Preemptive vs cooperative scheduling

| | Preemptive | Cooperative |
|--|-----------|------------|
| **How** | Scheduler can interrupt a task at any time | Task voluntarily yields the CPU |
| **Guarantee** | High-priority task always runs within bounded time | No guarantee — one task can starve others |
| **Used in** | FreeRTOS, Zephyr, RT Linux | Early embedded systems, simple state machines |
| **Risk** | Race conditions on shared data | One misbehaving task blocks everything |

**Almost all modern RTOSes are preemptive.** FreeRTOS is preemptive by default.

---

## Key RTOS concepts

### Tasks / Threads
Each task is an infinite loop with a priority. Higher priority = runs first when ready.
```c
void myTask(void *pvParameters) {
    while(1) {
        // do work
        vTaskDelay(pdMS_TO_TICKS(10));  // yield for 10ms
    }
}
xTaskCreate(myTask, "MyTask", 128, NULL, 3, NULL);  // priority 3
```

### Scheduler states
```
Running → Blocked (waiting for event/delay)
Running → Ready (preempted by higher priority)
Ready   → Running (scheduler picks it)
Blocked → Ready  (event fires)
```

### Tick / time quantum
- RTOS runs a timer interrupt at fixed rate (tick rate, e.g. 1kHz = 1ms ticks)
- Scheduler runs at each tick, picks highest-priority ready task
- `vTaskDelay(10)` = yield for 10 ticks

---

## Priority inversion — the classic RTOS bug

**Scenario:**
1. Low-priority task L holds mutex M
2. High-priority task H preempts L, tries to acquire M → blocked
3. Medium-priority task Mx runs (it's ready) and starves L
4. H is stuck waiting because L can't finish

**Result:** H, the highest-priority task, is blocked by a medium-priority task. Mars Pathfinder bug (1997) was priority inversion.

**Fix: Priority inheritance**
When H blocks on M held by L, temporarily boost L's priority to H's level. L finishes, releases M, drops back to its original priority. H runs.

FreeRTOS mutexes use priority inheritance automatically. Binary semaphores do NOT.

---

## Semaphore vs Mutex

| | Semaphore | Mutex |
|--|-----------|-------|
| **Use** | Signaling between tasks | Mutual exclusion |
| **Owner** | None — any task can give/take | Owner = task that took it |
| **Priority inheritance** | No | Yes (FreeRTOS mutex) |
| **Deadlock risk** | No | Yes (if two mutexes acquired in different order) |

**Rule:** use mutex for shared resource protection, semaphore for event signaling.

---

## Deadline types

| | Hard real-time | Soft real-time | Firm real-time |
|--|---------------|---------------|---------------|
| **Miss deadline** | System failure | Degraded performance | Result useless but no failure |
| **Example** | ABS brakes, pacemaker | Video streaming | Sensor data (stale = ignored) |
| **Guarantee** | Mathematically proven WCET | Statistical bounds | |

Robotics control loops are **hard real-time** for safety-critical paths (braking, collision avoidance), **soft real-time** for navigation.

---

## Schedulability analysis (RMS)

Rate Monotonic Scheduling: assign priority based on period (shorter period = higher priority). A set of n periodic tasks is schedulable if:

```
U = Σ(Cᵢ/Tᵢ) ≤ n(2^(1/n) - 1)
```

Where Cᵢ = execution time, Tᵢ = period. For large n, limit → ln(2) ≈ 0.693.

If total CPU utilization < 69.3%, RMS guarantees all deadlines are met.

---

## Links
- Related: [[FreeRTOS]], [[Real-Time Determinism]], [[AVR Peripherals]], [[Watchdog Timers]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is priority inversion and how does priority inheritance fix it? ? Priority inversion: low-priority task holds a mutex that a high-priority task needs, while a medium-priority task runs and starves the low-priority task, blocking the high-priority task indefinitely. Priority inheritance: temporarily boost the mutex holder's priority to match the waiting high-priority task so it can finish and release.

Mutex vs semaphore — when do you use each? ? Mutex: mutual exclusion on a shared resource — has an owner, supports priority inheritance. Semaphore: signaling between tasks — no owner, any task can give/take. Use mutex to protect shared data, semaphore to signal that an event occurred.

RMS schedulability bound for large n? ? U = Σ(Cᵢ/Tᵢ) ≤ ln(2) ≈ 0.693. If total CPU utilization across all periodic tasks is below 69.3%, Rate Monotonic Scheduling guarantees all deadlines are met.

Preemptive vs cooperative scheduling — key difference? ? Preemptive: scheduler interrupts any task at any tick to run a higher-priority ready task — bounded response time guaranteed. Cooperative: tasks yield voluntarily — one misbehaving task starves everything else. All modern RTOSes are preemptive.
