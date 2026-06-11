---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: tags: [ros2, lifecycle, composition]

# ROS2 Node & Lifecycle

> [!question] Explain it cold
> 
> - What is a node, and what's the difference between a regular node and a lifecycle (managed) node?
> - Walk the lifecycle state machine.
> - Why does composition matter for a real-time system?

---

## Core idea

A node is the unit of computation — it owns publishers, subscribers, services, timers, parameters. A **lifecycle (managed) node** adds an explicit state machine so an orchestrator can bring subsystems up/down in a controlled, deterministic order instead of everything racing to life on launch.

## Key facts & formulas

- **Lifecycle states**: `Unconfigured → Inactive → Active → Finalized`, plus transition states (Configuring, Activating, Deactivating, CleaningUp, ShuttingDown, ErrorProcessing).
- **Transitions**: `configure` (alloc resources, set up pubs/subs but don't publish), `activate` (start processing/publishing), `deactivate` (stop publishing, keep resources), `cleanup` (release back to Unconfigured), `shutdown`.
- Key property: in **Inactive**, the node exists and is configured but produces no output. Lets you stage a system — configure everything, then activate in dependency order.
- **Composition**: multiple nodes loaded into a single process (component container) → intra-process comms can skip serialization entirely (pointer passing). This is the ROS2 answer to the ROS1 nodelet.
- Intra-process zero-copy is the "free" latency win before you reach for [[iceoryx Zero-Copy]] (which handles inter-process).


## Interview follow-ups

- **Q:** Why use lifecycle nodes instead of just starting everything in `main()`?
    - **A:** Deterministic, orchestrated bring-up and teardown. You configure all nodes (allocate, wire up) then activate in dependency order, and can deactivate to a safe no-output state without destroying the node. Critical for safety-critical or restart-without-relaunch scenarios.
- **Q:** What does composition buy you?
    - **A:** Nodes in one process can pass messages intra-process by pointer — no serialization, no loopback networking. Big latency/CPU win for high-rate pipelines.
- **Q:** Difference between lifecycle Active and your app's "running" state?
    - **A:** Lifecycle is a framework-level managed-node concept; an app state machine (like WALL-E's modes) is your own logic. Don't conflate them.


## Links

- Related: [[ROS2 Executors]], [[iceoryx Zero-Copy]], [[colcon ament & launch]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

The four primary lifecycle node states? ? Unconfigured, Inactive, Active, Finalized (with transition states between them).

What does configure do vs activate in a lifecycle node? ? configure allocates resources and sets up pubs/subs but produces no output; activate starts actual processing and publishing.

What latency win does node composition provide? ? Nodes in one process can pass messages intra-process by pointer — no serialization or loopback networking.