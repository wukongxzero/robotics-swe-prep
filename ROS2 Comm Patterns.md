---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: 2026-06-02 tags: [ros2, topics, services, actions]

# ROS2 Comm Patterns


---

## Core idea

Three communication primitives, chosen by _interaction shape_: **topics** (continuous many-to-many streaming), **services** (synchronous request/response), **actions** (long-running goals with feedback + cancellation). All three sit on top of DDS pub/sub.

## Key facts & formulas

- **Topics** — async, decoupled, many-to-many. No reply. Best for sensor streams, state broadcasts, commands. Governed by [[ROS2 QoS]].
- **Services** — sync-ish request/response, one server. Best for quick queries / config changes that need a confirmation ("set mode", "get param"). Blocking the caller on a long service is an anti-pattern.
- **Actions** — for goals that take time and need: periodic **feedback**, a final **result**, and **cancellation/preemption**. Built _from_ topics + services under the hood (goal service, result service, feedback topic, status topic, cancel service).
- Decision rule: _streaming?_ → topic. _Quick call-and-answer?_ → service. _Takes seconds+, want progress and the ability to cancel?_ → action.


## Links

- Related: [[ROS2 QoS]], [[ROS2 Executors]], [[Nav2 Stack]]
- Parent: [[00 Knowledge Map]]

---

