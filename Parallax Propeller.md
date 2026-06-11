---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, parallax, propeller, multicore, kapila-final]

# Parallax Propeller

> [!question] Explain it cold
> 
> - What's the defining architectural feature of the Propeller?
> - dira / outa / ina — what are they?
> - Why does the 8-cog model give you deterministic timing without interrupts?

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

## Where I've used it / context

- **Kapila final**: covered the 8-cog model, dira/outa/ina, cog_run alongside the AVR and Jetson material. The contrast with AVR is the teaching point — AVR multiplexes with interrupts and timers; Propeller dedicates a core per task.

## Interview follow-ups

- **Q:** Propeller has no interrupts — how do you do real-time then?
    - **A:** Dedicate a cog to the time-critical task. It runs a deterministic tight loop on its own core with no preemption, so timing is rock-solid without interrupt prioritization.
- **Q:** How do cogs share data safely?
    - **A:** Through hub RAM via the time-sliced round-robin hub window; the deterministic access pattern is what keeps it predictable. Locks exist for coordination.
- **Q:** dira/outa/ina vs DDRx/PORTx/PINx?
    - **A:** Same conceptual roles (direction / output / input) but per-cog, and pin outputs are OR-combined across all cogs at the hub.

## Gotchas / what trips me up

- Calling it interrupt-driven — the whole point is it _isn't_.
- Forgetting outputs are OR-combined across cogs (two cogs driving the same pin interact).

## Links

- Related: [[AVR Register Programming]], [[Real-Time Determinism]], [[Compute & acceleration]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What's the defining feature of the Parallax Propeller? ? 8 independent cores (cogs) and no interrupts — real-time tasks each get a dedicated cog for deterministic timing.

Propeller's dira / outa / ina? ? Per-cog direction, output, and input registers — the Propeller analog of AVR's DDRx/PORTx/PINx (pin outputs OR-combined across cogs at the hub).

How does the Propeller achieve determinism without interrupts? ? Dedicate a cog to each time-critical task; it runs an unpreempted tight loop on its own core, so no interrupt contention or jitter.