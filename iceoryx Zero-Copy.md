---
type: concept
domain: ROS2 & middleware
status: drafted
last-reviewed: 2026-05-27
tags: [ros2, iceoryx, zero-copy, real-time, differentiator]
---

# iceoryx Zero-Copy

> [!star] Differentiator territory
> Most RSE candidates have *used* ROS2 topics. Far fewer have run **iceoryx zero-copy over DDS in production** on a latency-critical control path. This note should be the most fluent one in the vault — it's the Articulus story that separates you.


---

## Core idea

iceoryx is a **shared-memory, true zero-copy** inter-process transport. Instead of serializing a message, copying it through the network stack, and deserializing on the other side, the publisher writes data **once** into a shared-memory segment; subscribers read the *same physical memory* by reference. The payload is never copied — latency is constant regardless of message size.

---

## Key facts & formulas

### The mechanism end-to-end

```
Publisher process          RouDi daemon           Subscriber process
      |                        |                        |
  loan() ─────────────────────►|                        |
      |◄── chunk ptr ──────────|                        |
  write into chunk             |                        |
  publish(chunk) ─────────────►|                        |
      |              notify subscriber via              |
      |              lock-free queue ─────────────────►|
      |                        |              take() ──►|
      |                        |      ◄── same chunk ptr|
      |                        |      read in place     |
      |                        |      release() ────────|
```

**RouDi** (Routing and Discovery) is the daemon. It:
- Owns and manages the **shared-memory segments** (POSIX `shm_open`)
- Brokers pub/sub connections and service discovery
- Cleans up leaked chunks if a process dies
- Does **not** sit in the data path — data moves directly via the shared segment

### Publisher API (typed, C++)

```cpp
// Offer the topic
auto publisher = iox::popo::Publisher<RadiusMessage>({"Surgical", "ArmState", "v1"});
publisher.offer();

// Per-message cycle
publisher.loan()
    .and_then([](auto& sample) {
        sample->x = 1.0f;
        sample->y = 2.0f;
        sample.publish();          // zero-copy: just enqueues the pointer
    })
    .or_else([](auto& error) { /* handle pool exhaustion */ });
```

`loan()` returns `expected<Sample, AllocationError>` — you *must* handle the error case (memory pool can be exhausted).

### Subscriber API (typed, C++)

```cpp
auto subscriber = iox::popo::Subscriber<RadiusMessage>({"Surgical", "ArmState", "v1"});
subscriber.subscribe();

// In callback/poll loop
subscriber.take()
    .and_then([](const auto& sample) {
        use(sample->x);            // reading shared memory directly
    })                             // chunk auto-released when sample goes out of scope
    .or_else([](auto& error) { /* no data or error */ });
```

Chunk is **reference-counted** — released back to the pool automatically when the last subscriber's `sample` destructor runs.

### Typed vs untyped API

| | Typed API | Untyped API |
|---|---|---|
| Access | `iox::popo::Publisher<T>` | `iox::popo::UntypedPublisher` |
| Style | C++ idiomatic, type-safe | void* based |
| Use case | Application code | Framework integration (e.g. rmw_iceoryx) |

### WaitSet vs Listener (avoid busy-polling)

- **WaitSet** (reactor pattern): attach multiple publishers/subscribers, call `wait()` — blocks until any event fires, returns a notification vector. Single thread handles many objects.
- **Listener** (proactor pattern): attach a callback that fires in a background thread when an event arrives. Fire-and-forget style.

Both use non-busy waits (no spinning). Don't poll `take()` in a loop — use one of these.

### n:m communication

Unlike DDS with its per-connection copies, iceoryx supports **n publishers : m subscribers** on the same topic — all m subscribers read the *same chunk*, ref-counted. One write feeds any number of readers with no additional copies.

### ROS2 integration

Two paths:
1. **`rmw_iceoryx`**: swap the RMW layer — `RMW_IMPLEMENTATION=rmw_iceoryx_cpp ros2 run ...`. Uses the untyped API internally. DDS handles discovery; iceoryx handles the data transport. Requires fixed-size message types.
2. **Loaned messages** (Fast DDS data-sharing): similar concept, stays within DDS — `borrow_loaned_message()` → write → `publish()`. Less control than full iceoryx but no extra daemon.

### Starting RouDi

```bash
# Standalone
iox-roudi

# With custom memory config (pool sizes)
iox-roudi -c /path/to/roudi_config.toml
```

RouDi must be running before any publisher/subscriber starts. In production, run it as a systemd service.

### CMake integration

```cmake
find_package(iceoryx_posh CONFIG REQUIRED)
find_package(iceoryx_hoofs CONFIG REQUIRED)

target_link_libraries(my_node
    PRIVATE iceoryx_posh::iceoryx_posh
            iceoryx_hoofs::iceoryx_hoofs)
```

### Constraints (the first thing a sharp interviewer probes)

| Constraint | Why |
|---|---|
| Fixed-size / POD-like messages | Layout must be valid across process boundaries in shared memory — no internal heap pointers |
| Same host only | It's POSIX shared memory, not a network transport |
| 64-bit hardware | Memory addressing requirement |
| RouDi must be running | It bootstraps the shared-memory segments |

---


---


---


---

## Links
- Related: [[DDS & RMW]], [[Latency Budgets]], [[Safety-Critical Architecture]], [[ROS2 Node & Lifecycle]], [[Real-Time Determinism]], [[ROS2 Comm Patterns]]
- Parent: [[00 Knowledge Map]]

---

