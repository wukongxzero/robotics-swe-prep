---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: 2026-06-02 tags: [ros2, qos, dds, real-time]

# ROS2 QoS


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


## Links

- Related: [[ROS2 Executors]], [[DDS & RMW]], [[ROS2 Comm Patterns]]
- Parent: [[00 Knowledge Map]]

---

