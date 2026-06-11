---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: tags: [ros2, lifecycle, composition]

# ROS2 Node & Lifecycle


---

## Core idea

A node is the unit of computation — it owns publishers, subscribers, services, timers, parameters. A **lifecycle (managed) node** adds an explicit state machine so an orchestrator can bring subsystems up/down in a controlled, deterministic order instead of everything racing to life on launch.

## Key facts & formulas

- **Lifecycle states**: `Unconfigured → Inactive → Active → Finalized`, plus transition states (Configuring, Activating, Deactivating, CleaningUp, ShuttingDown, ErrorProcessing).
- **Transitions**: `configure` (alloc resources, set up pubs/subs but don't publish), `activate` (start processing/publishing), `deactivate` (stop publishing, keep resources), `cleanup` (release back to Unconfigured), `shutdown`.
- Key property: in **Inactive**, the node exists and is configured but produces no output. Lets you stage a system — configure everything, then activate in dependency order.
- **Composition**: multiple nodes loaded into a single process (component container) → intra-process comms can skip serialization entirely (pointer passing). This is the ROS2 answer to the ROS1 nodelet.
- Intra-process zero-copy is the "free" latency win before you reach for [[iceoryx Zero-Copy]] (which handles inter-process).


## Links

- Related: [[ROS2 Executors]], [[iceoryx Zero-Copy]], [[colcon ament & launch]]
- Parent: [[00 Knowledge Map]]

---

