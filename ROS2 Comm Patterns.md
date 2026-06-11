---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: 2026-06-02 tags: [ros2, topics, services, actions]

# ROS2 Comm Patterns

> [!question] Explain it cold
> 
> - The three patterns and the one-line decision rule for each.
> - When is an action the right call over a service?
> - What's actually under the hood for all three?

---

## Core idea

Three communication primitives, chosen by _interaction shape_: **topics** (continuous many-to-many streaming), **services** (synchronous request/response), **actions** (long-running goals with feedback + cancellation). All three sit on top of DDS pub/sub.

## Key facts & formulas

- **Topics** — async, decoupled, many-to-many. No reply. Best for sensor streams, state broadcasts, commands. Governed by [[ROS2 QoS]].
- **Services** — sync-ish request/response, one server. Best for quick queries / config changes that need a confirmation ("set mode", "get param"). Blocking the caller on a long service is an anti-pattern.
- **Actions** — for goals that take time and need: periodic **feedback**, a final **result**, and **cancellation/preemption**. Built _from_ topics + services under the hood (goal service, result service, feedback topic, status topic, cancel service).
- Decision rule: _streaming?_ → topic. _Quick call-and-answer?_ → service. _Takes seconds+, want progress and the ability to cancel?_ → action.

## Where I've used it

- **WALL-E V3**: TankStatus / odometry / IMU = topics (streams). "Switch to AUTONOMOUS" = service-shaped (confirmed state change). A "navigate to landmark" behavior is the textbook action — long-running, want feedback, want to cancel. (Nav2 exposes navigation as actions — relevant for the summer Nav2 work.)
- **FENCE-BOT**: `/vr_pose` is a topic (continuous teleop stream). A "home the arm" command would be an action.
- **Articulus**: high-rate control/telemetry over topics; mode/config changes as services; anything long-horizon as an action so it's cancellable mid-execution — important for safety.

## Interview follow-ups

- **Q:** Why not just use a service for "move to pose"?
    - **A:** A service blocks the caller and gives no progress or cancellation. Motion takes time and may need to be aborted — that's exactly what actions provide (feedback, result, cancel).
- **Q:** Actions are built from what?
    - **A:** Topics + services: a goal service, a cancel service, a result service, plus feedback and status topics. They're a convention on top of the primitives, not a separate transport.
- **Q:** Many subscribers to one topic — issue?
    - **A:** Fine, that's the design (many-to-many), but mind QoS compatibility per pair and the publisher's serialization cost if not intra-process.

## Gotchas / what trips me up

- Reaching for a service on a long task → blocks the executor thread, no cancel. Use an action.
- Forgetting actions are composite — debugging them means looking at the underlying topics/services.

## Links

- Related: [[ROS2 QoS]], [[ROS2 Executors]], [[Nav2 Stack]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

One-line decision rule for topic vs service vs action? ? Streaming → topic. Quick request/response → service. Long-running goal needing feedback and cancellation → action.

What are ROS2 actions built from under the hood? ? Topics + services: goal service, cancel service, result service, plus feedback and status topics.

Why is a service the wrong choice for a "move to pose" command? ? It blocks the caller, gives no progress feedback, and can't be cancelled mid-execution — all things an action provides.