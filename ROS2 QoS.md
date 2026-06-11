---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: 2026-06-02 tags: [ros2, qos, dds, real-time]

# ROS2 QoS

> [!question] Explain it cold
> 
> - What is a QoS profile and what problem does it solve?
> - Name the policies and what each controls.
> - When does a publisher/subscriber pair _fail to connect_ even though both exist?

---

## Core idea

QoS (Quality of Service) is the contract that governs _how_ messages are delivered on a topic — reliability, persistence, ordering depth, timing — at the DDS layer. A publisher and subscriber only communicate if their QoS profiles are **compatible**; mismatched profiles silently drop the connection.

## Key facts & formulas

- **Reliability**
    - `RELIABLE` — retransmits until delivered (TCP-like). Use for commands, configs.
    - `BEST_EFFORT` — fire and forget (UDP-like). Use for high-rate sensor streams where the freshest sample matters more than every sample.
- **Durability**
    - `VOLATILE` — late subscribers get nothing.
    - `TRANSIENT_LOCAL` — publisher keeps last N samples for late joiners (latched-topic behavior). Standard for `/tf_static`, maps, robot_description.
- **History**
    - `KEEP_LAST` with `depth=N` — ring buffer of the last N. `depth=1` = only freshest.
    - `KEEP_ALL` — bounded only by resource limits.
- **Deadline** — max expected gap between messages. Missed → callback fires. Watchdog at the QoS layer.
- **Liveliness** — how a publisher asserts it's alive (`AUTOMATIC` vs `MANUAL_BY_TOPIC`) + lease duration.
- **Lifespan** — how long a sample stays valid before expiry.

## Compatibility rule (the trap)

Subscriber's _requested_ QoS must be **no stronger** than the publisher's _offered_ QoS:

- Pub `BEST_EFFORT` + Sub `RELIABLE` → **incompatible**, no data.
- Pub `RELIABLE` + Sub `BEST_EFFORT` → compatible (sub asked for less).
- Same logic for durability and deadline. "Requesting more than is offered" breaks the match.

## Where I've used it

- **Articulus**: control commands on `RELIABLE` + `depth=1` (latest command, guaranteed); high-rate telemetry on `BEST_EFFORT` so a dropped sample never stalls the loop. This is the QoS-mirror of the executor grouping decision.
- **WALL-E V3**: odometry / IMU streams want `BEST_EFFORT depth=1` — I only care about the newest pose, not a backlog. State-machine mode commands want `RELIABLE`.
- **FENCE-BOT**: `/vr_pose` at 50 Hz is a freshness-over-completeness stream → `BEST_EFFORT`, `KEEP_LAST depth=1`.

## Interview follow-ups

- **Q:** Your sensor publisher is running but the subscriber callback never fires. QoS is the suspect — what's the first thing you check?
    - **A:** Reliability/durability compatibility. Classic case: sensor published `BEST_EFFORT`, subscriber defaulted to `RELIABLE`. Use `ros2 topic info -v` to see offered vs requested.
- **Q:** Why `depth=1` for a 200 Hz IMU?
    - **A:** I want the freshest sample, not a queue that grows under backpressure and feeds me stale data. Deep queues add latency for streaming sensors.
- **Q:** What's `TRANSIENT_LOCAL` for?
    - **A:** Latching — a late-joining subscriber immediately gets the last published value. Used for static transforms, maps, descriptions that don't republish.

## Gotchas / what trips me up

- Forgetting compatibility is _directional_ — sub can ask for less than pub offers, never more.
- Using `sensor_data` QoS preset blindly (it's BEST_EFFORT) then wondering why a RELIABLE subscriber gets nothing.
- Confusing QoS (delivery contract) with the executor (threading). Different layers — see [[ROS2 Executors]].

## Links

- Related: [[ROS2 Executors]], [[DDS & RMW]], [[ROS2 Comm Patterns]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

QoS compatibility rule between publisher and subscriber? ? The subscriber's requested QoS must be no stronger than the publisher's offered QoS. Sub can ask for less, never more.

Pub BEST_EFFORT + Sub RELIABLE — do they communicate? ? No. The subscriber requested a stronger guarantee than the publisher offers — incompatible, no data flows.

Which QoS settings for a 200 Hz IMU stream and why? ? BEST_EFFORT, KEEP_LAST, depth=1 — freshest sample matters more than every sample, and a shallow queue avoids stale-data latency under backpressure.

What does TRANSIENT_LOCAL durability give you? ? Latching — late-joining subscribers immediately receive the last published sample. Used for maps, static transforms, robot_description.