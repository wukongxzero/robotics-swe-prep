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

> [!question] Explain it cold
> *Before reading anything below, say or write the answer from memory.*
>
> - What is zero-copy and what cost does it eliminate?
> - How does iceoryx achieve it (the actual mechanism)?
> - What are the API calls a publisher makes, in order?
> - When would you NOT use it?

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

## Where I've used it

- **Articulus Surgical**: primary architect of the real-time control stack for the multi-arm laparoscopic system. Ran **iceoryx over DDS** specifically to hit the latency budget on the control path — zero-copy meant high-rate state/command data moved between processes without serialize/copy tax, keeping end-to-end loop timing deterministic. This is the concrete reason the surgical system met its latency SLA.
- Ties directly to [[Latency Budgets]] and [[Safety-Critical Architecture]] — low, *predictable* latency is a safety property in teleoperated surgery, not just a performance win.

---

## Interview follow-ups

- **Q: What is "zero" in zero-copy?**
  - **A:** Zero *copies* of the payload between producer and consumer. Publisher writes once into shared memory; all subscribers read that same memory. Eliminates: serialize → copy → transport → deserialize.

- **Q: Why is latency constant with payload size?**
  - **A:** You transfer a pointer/handle into the shared-memory chunk, not the bytes. A 1 KB and a 10 MB message take the same time to "send" — nothing is moved.

- **Q: What's RouDi and what does it do at runtime?**
  - **A:** The iceoryx daemon — Routing and Discovery. It allocates and manages the POSIX shared-memory segments and brokers pub/sub connections. Critically, it is *not* in the hot data path — the data moves directly between publisher and subscriber via the segment; RouDi only manages the bookkeeping and cleanup.

- **Q: Walk me through the publisher API calls.**
  - **A:** `offer()` → per-message: `loan()` gets a chunk pointer, write into it, `publish()` enqueues the pointer for subscribers. If `loan()` returns an error (pool exhausted), handle it — don't just ignore `expected<>`.

- **Q: How does a subscriber avoid busy-polling?**
  - **A:** Use a WaitSet (blocks until any attached object has data, reactor pattern) or a Listener (attaches a callback fired in a background thread). Never spin on `take()` in a loop.

- **Q: Why can't you use iceoryx for cross-machine communication?**
  - **A:** It's POSIX shared memory — a hardware-level shared address space. That only exists within a single host. For cross-machine you'd still use DDS (with UDP/TCP); iceoryx handles the on-host hot path.

- **Q: What constraint does iceoryx impose on message types, and why?**
  - **A:** Fixed-size, POD-like — no internal heap pointers. The memory layout must be interpretable by any process that maps the segment; a pointer valid in one process's heap is garbage in another's address space.

- **Q: When would you NOT use iceoryx?**
  - **A:** Cross-machine comms, variable-length/heap-heavy messages (e.g. `std::vector` inside the message), low-rate small messages where copy cost is negligible and RouDi operational overhead isn't worth it, or anywhere you can't guarantee RouDi is running.

---

## Gotchas / what trips me up

- Calling it "faster networking" — it is *not* networking. It's POSIX shared memory on one host.
- Forgetting the POD/fixed-size constraint — first thing a sharp interviewer probes.
- Forgetting to handle `expected<>` errors from `loan()` — pool can be exhausted if subscribers hold chunks too long.
- Conflating intra-process zero-copy (same process, [[ROS2 Node & Lifecycle]] composition) with iceoryx (separate processes, shared memory). Both avoid copies, different scope and mechanism.
- RouDi must be running before any node starts — "connection refused" at startup means RouDi isn't up.
- `release()` on a subscriber chunk is automatic via RAII, but if you copy the raw pointer out of the `sample` scope, you've broken the ref count.

---

## Links
- Related: [[DDS & RMW]], [[Latency Budgets]], [[Safety-Critical Architecture]], [[ROS2 Node & Lifecycle]], [[Real-Time Determinism]], [[ROS2 Comm Patterns]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does iceoryx zero-copy eliminate from the message path?
?
Serialization, the payload copy, transport, and deserialization — the publisher writes once into shared memory and all subscribers read that same chunk by reference.
<!--SR:!2026-05-27,0,230-->

Why is iceoryx transfer latency independent of message size?
?
You transfer a pointer to a shared-memory chunk, not the bytes — so payload size doesn't affect send time.
<!--SR:!2026-05-27,0,230-->

What is RouDi and what does it NOT do?
?
RouDi (Routing and Discovery) manages POSIX shared-memory segments and brokers pub/sub connections. It is NOT in the hot data path — data moves directly between processes; RouDi only handles bookkeeping and cleanup.
<!--SR:!2026-05-28,1,230-->

Walk through the publisher API calls in order.
?
`offer()` once; then per-message: `loan()` → write into chunk → `publish()`. Handle the `expected<>` error from `loan()` (pool can exhaust).
<!--SR:!2026-05-27,0,230-->

What are WaitSet and Listener, and why do you need them?
?
WaitSet (reactor): blocks until any attached object has data, returns a notification vector — one thread handles many topics. Listener (proactor): fires a callback in a background thread per event. Both avoid busy-polling `take()`.

What constraints does iceoryx impose on message types and deployment?
?
Fixed-size / POD-like (no internal heap pointers — layout must be valid across process address spaces), and same-host only (POSIX shared memory, not network transport).

How does rmw_iceoryx integrate iceoryx into ROS2?
?
Swaps the RMW layer: `RMW_IMPLEMENTATION=rmw_iceoryx_cpp`. DDS handles discovery; iceoryx handles the actual data movement via its untyped API. Requires fixed-size message types.

When would you NOT use iceoryx?
?
Cross-machine comms, variable-length/heap-heavy messages, low-rate small messages where copy cost is negligible, or when you can't guarantee RouDi is running.
