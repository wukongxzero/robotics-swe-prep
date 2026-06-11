---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, parallax, propeller, multicore, kapila-final]

# Parallax Propeller


---

## Core idea

The Parallax Propeller is an 8-core ("cog") microcontroller with **no interrupts by design** — instead of one CPU juggling tasks via interrupts, you dedicate a cog to each real-time task. Determinism comes from physical parallelism, not interrupt prioritization.

## Key facts & formulas

- **8 cogs**: 8 identical 32-bit cores sharing a hub. Each runs independently and deterministically.
- **Hub access**: cogs access shared hub RAM in a round-robin time-sliced manner (the "hub window"), which is what makes timing predictable.
- **Per-cog I/O registers** (the parallel of AVR's DDR/PORT/PIN):
    - **dira** — direction register (1 = output) for that cog.
    - **outa** — output state register.
    - **ina** — input read register.
    - Pin states are OR-combined across cogs at the hub.
- **cog_run / cognew / coginit**: launch a task on a free cog. This is how you spin up a dedicated driver (e.g. a software UART, a PWM generator) on its own core.
- **No interrupts**: a task that needs precise timing gets its own cog running a tight loop — no preemption, no jitter from ISR contention.


## Links

- Related: [[AVR Register Programming]], [[Real-Time Determinism]], [[Compute & acceleration]]
- Parent: [[00 Knowledge Map]]

---

