---
type: concept
domain: SWE Foundations
status: drafted
last-reviewed: 2026-05-27
tags: [linux, ipc, threading, system-design, concurrency, debugging, swe]
---

# Linux, IPC & System Design

> [!question] Explain it cold
> *Before reading anything below, say or write the answer from memory.*
>
> - What's the difference between a process and a thread?
> - Name four IPC mechanisms and when you'd pick each one.
> - What is priority inversion, and how do you fix it?
> - Design a sensor-fusion node from scratch — what are the concurrency boundaries?

---

## Core idea

Robotics SWE interviews test three OS-level competencies beyond ROS2: **process/thread model**, **IPC mechanisms** (how data moves between processes), and **concurrency safety** (locks, priority, deadlock). System design questions then stack all of these into "design a robot that does X."

---

## Key facts & formulas

### Process vs. thread

| | Process | Thread |
|---|---|---|
| Memory | Own virtual address space | Shares address space with siblings |
| Isolation | Crash doesn't affect others | Crash takes down the whole process |
| Creation cost | High (`fork`/`exec`) | Low (`pthread_create` / `std::thread`) |
| Communication | IPC (pipes, sockets, shm) | Shared variables + synchronization |
| Use in robotics | Separate nodes (ROS2 model) | Concurrent tasks inside a node |

**Key**: ROS2 nodes are separate processes. Inside a node, callback groups run in threads. iceoryx bridges processes via shared memory (zero-copy, see [[iceoryx Zero-Copy]]).

### Linux scheduling for robotics

```bash
# Set a thread to SCHED_FIFO (real-time), priority 80
pthread_attr_setschedpolicy(&attr, SCHED_FIFO);
pthread_attr_setschedparam(&attr, {.sched_priority = 80});

# Or at runtime
struct sched_param sp = {.sched_priority = 80};
sched_setscheduler(0, SCHED_FIFO, &sp);

# Check what's running at RT priority
chrt -p <pid>
ps -eo pid,cls,rtprio,comm
```

- **CFS** (default): fair-share, not deterministic. Fine for perception/LLM.
- **SCHED_FIFO**: runs until it blocks or a higher-priority task preempts. Use for the control loop thread. Priority 1–99 (99 = highest).
- **SCHED_RR**: like FIFO but with a time-slice quantum — rarely needed in robotics.
- **RT_PREEMPT patch**: makes Linux fully preemptible, reducing worst-case latency from ~ms to ~tens of µs. Required for hard real-time on Linux. See [[Real-Time Determinism]].

### IPC mechanisms — the comparison table

| Mechanism | Speed | Scope | Use case |
|---|---|---|---|
| **Pipe / FIFO** | Medium | Same host | Shell pipelines, simple parent↔child |
| **Unix domain socket** | Fast | Same host | Bidirectional, stream or datagram |
| **TCP/UDP socket** | Variable | Network | Cross-machine, DDS uses this |
| **POSIX shared memory** | Fastest | Same host | Large data, zero-copy (iceoryx) |
| **Message queue (mq)** | Medium | Same host | Structured msgs with priority |
| **Signal** | N/A (notification only) | Same host | Async events, SIGTERM, SIGUSR1 |

Robotics pattern: **shared memory + semaphore/condvar for notification** = maximum throughput for sensor data. DDS = structured pub/sub over UDP. iceoryx = shared memory managed for you.

### Concurrency primitives

```cpp
// Mutex — mutual exclusion, one thread at a time
std::mutex mtx;
std::lock_guard<std::mutex> lk(mtx);  // RAII, auto-unlocks

// Condition variable — wait until a condition is true
std::condition_variable cv;
std::unique_lock<std::mutex> ul(mtx);
cv.wait(ul, []{ return data_ready; });   // atomically releases lock & sleeps
// ...producer side:
{ std::lock_guard<std::mutex> lk(mtx); data_ready = true; }
cv.notify_one();

// Atomic — lock-free single-variable ops
std::atomic<bool> stop_flag{false};
stop_flag.store(true, std::memory_order_release);
if (stop_flag.load(std::memory_order_acquire)) { ... }

// Semaphore (C++20)
std::counting_semaphore<10> sem(0);
sem.release();    // signal (V)
sem.acquire();    // wait  (P)
```

**Mutex vs semaphore**: mutex has ownership (only the locker unlocks it, prevents priority inversion via PI mutex). Semaphore has no ownership — good for signaling between threads (producer releases, consumer acquires).

### Priority inversion — the robotics killer

**What it is**: A low-priority task holds a lock; a high-priority task blocks waiting for it; a medium-priority task preempts the low-priority one. High-priority starves indefinitely. (Mars Pathfinder, 1997.)

**Solutions**:
1. **Priority Inheritance** (`PTHREAD_MUTEX_PROTOCOL = PTHREAD_PRIO_INHERIT`): OS temporarily boosts the lock-holder to the waiter's priority. Resolves automatically.
2. **Priority Ceiling** (`PTHREAD_PRIO_PROTECT`): mutex has a fixed ceiling; any thread that locks it runs at ceiling priority. Simpler but requires knowing the highest user priority upfront.
3. **Lock-free data structures**: eliminate the lock entirely (ring buffer with atomics).

```cpp
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mtx, &attr);
```

### Deadlock — four conditions (all must hold)

1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait

**Break it**: always acquire locks in the same global order (breaks circular wait). Or use `std::lock(m1, m2)` which uses deadlock-avoidance internally.

### Lock-free ring buffer (producer/consumer)

```cpp
template<typename T, size_t N>
struct RingBuffer {
    std::array<T, N> buf;
    std::atomic<size_t> head{0}, tail{0};

    bool push(const T& v) {
        size_t h = head.load(std::memory_order_relaxed);
        size_t next = (h + 1) % N;
        if (next == tail.load(std::memory_order_acquire)) return false; // full
        buf[h] = v;
        head.store(next, std::memory_order_release);
        return true;
    }
    bool pop(T& v) {
        size_t t = tail.load(std::memory_order_relaxed);
        if (t == head.load(std::memory_order_acquire)) return false; // empty
        v = buf[t];
        tail.store((t + 1) % N, std::memory_order_release);
        return true;
    }
};
```

This is the pattern for sensor→control thread data on a real-time path: no malloc, no locks, O(1).

### Memory: virtual memory & cache

- Each process has a **virtual address space** (48-bit on x86_64 = 256 TB). The OS maps virtual pages to physical pages on demand.
- **Page fault**: accessing an unmapped page traps into the kernel to allocate. Unacceptable on a real-time thread. Fix: `mlockall(MCL_CURRENT | MCL_FUTURE)` locks all pages into RAM.
- **Cache hierarchy**: L1 (~4 cycle), L2 (~12 cycle), L3 (~40 cycle), DRAM (~200 cycle). Cache misses dominate latency in tight loops. Keep hot data in a contiguous struct, not scattered pointers.
- **False sharing**: two threads write different fields that land in the same 64-byte cache line — each write invalidates the other's cache entry. Fix: `alignas(64)` between fields.

```cpp
struct alignas(64) SensorState { float x, y, z; };  // pad to cache line
```

### Debugging tools — quick reference

| Tool | What it does |
|---|---|
| `htop` / `btop` | Interactive process monitor — CPU per-core, memory, load |
| `gdb` | Breakpoints, watchpoints, backtrace, attach to running process |
| `valgrind --tool=memcheck` | Heap memory errors (use-after-free, leaks). Slow — use on test builds |
| `valgrind --tool=helgrind` | Data race detection |
| `perf stat` | CPU cycles, cache misses, branch mispredicts for a run |
| `perf record/report` | Flame-graph-ready profiling of where CPU time goes |
| `strace -p <pid>` | System call trace — see every kernel call in real time |
| `lttng` / `trace-cmd` | Kernel tracing, scheduling latency measurement |
| `cyclictest` | Measure scheduling jitter on RT_PREEMPT kernels |
| `numactl` | NUMA topology — bind process to CPU/memory node |
| `taskset` | Pin process or thread to specific CPU cores |
| `iostat` / `vmstat` | Disk I/O and VM stats — is logging or storage blocking? |
| `ss` / `netstat` | Socket state — DDS connections, dropped packets |
| `lsof` | List open files/sockets by process |

---

### htop / btop — robotics use

**Install:**
```bash
sudo apt install htop btop
```

`btop` is the modern replacement — color-coded, GPU support, more readable. Use `htop` when btop isn't available.

**What to look at in a robotics context:**

```
F6 → sort by CPU%    # find which node is hogging a core
F5 → tree view       # see ROS2 nodes as process tree under ros2 run
u  → filter by user  # isolate your robot processes from system noise
```

**btop-specific:**
- Press `p` to toggle per-process CPU graphs
- GPU panel shows Jetson/CUDA utilization alongside CPU — useful for perception nodes
- Network panel: watch for DDS traffic spikes correlated with callback latency

**Key things to look for:**

| Symptom | What to check in htop/btop |
|---------|--------------------------|
| Control loop missing deadlines | Is a core at 100%? Which process? |
| ROS2 node not responding | Is it stuck (D state) or spinning (R state)? |
| System feels sluggish | Load average > core count → CPU-bound |
| Memory growing over time | RES column — memory leak in a node |
| Jetson thermal throttle | CPU frequency dropping in btop's CPU panel |

**Process states (the S column in htop):**
- `R` — running or runnable
- `S` — sleeping, waiting for I/O or timer (normal)
- `D` — uninterruptible sleep (kernel I/O) — stuck here = bad
- `Z` — zombie (child finished, parent hasn't `wait()`ed)

---

### perf — CPU profiling

```bash
# Profile a ROS2 node's CPU hotspots
perf record -g ros2 run my_pkg my_node
perf report --stdio

# One-liner stat — cycles, cache misses, branch mispredicts
perf stat -e cycles,cache-misses,branch-misses ./my_binary

# Record and generate a flamegraph (requires FlameGraph scripts)
perf record -F 99 -g -p <pid>
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > flame.svg
```

**Robotics use:** find which function inside a ROS2 node consumes the most CPU — common culprits are serialization in callbacks, point cloud processing, or accidentally running on the wrong thread.

---

### strace — syscall tracing

```bash
# Attach to a running ROS2 node by PID
strace -p <pid>

# Count syscalls — find unexpected write/read/futex calls
strace -c ros2 run my_pkg my_node

# Trace only specific syscalls
strace -e trace=read,write,futex -p <pid>

# Trace with timestamps
strace -T -tt -p <pid>
```

**Robotics use:** catch unexpected syscalls on a supposedly real-time path — `futex` calls mean lock contention, `mmap` means allocations, `write` means logging on a hot path.

---

### CPU affinity & NUMA — high performance

**taskset** — pin a process to a core:
```bash
# Run a process on core 2 only
taskset -c 2 ros2 run my_pkg my_node

# Pin an already-running PID to cores 2-3
taskset -cp 2,3 <pid>

# Check current affinity
taskset -p <pid>
```

**numactl** — NUMA-aware memory allocation (multi-socket systems, Jetson AGX):
```bash
# Show NUMA topology
numactl --hardware
numactl --show

# Run on NUMA node 0, allocate memory locally
numactl --cpunodebind=0 --membind=0 ./my_node

# Check NUMA memory stats
numastat -p <pid>
```

**Why it matters:** on a Jetson AGX or multi-socket server, accessing memory on the "wrong" NUMA node costs 2–4× more latency. Bind your control-loop process to the NUMA node closest to its memory.

**Robotics HPC pattern:**
```bash
# Isolate a CPU core from the kernel scheduler entirely (kernel boot param)
# Add to /etc/default/grub: GRUB_CMDLINE_LINUX="isolcpus=3 nohz_full=3 rcu_nocbs=3"
# Then dedicate that core to your RT thread
taskset -c 3 my_control_loop
```

---

### gdb — debugging a running ROS2 node

```bash
# Attach to a running node
gdb -p <pid>

# Run with debug symbols
gdb --args ros2 run my_pkg my_node

# Common commands inside gdb
(gdb) bt           # backtrace — where am I?
(gdb) info threads # list all threads
(gdb) thread 3     # switch to thread 3
(gdb) bt           # backtrace for that thread
(gdb) watch var    # break when var changes
(gdb) p var        # print variable value
(gdb) c            # continue
(gdb) finish       # run until current function returns
```

**Robotics use:** attach to a crashed or hung node post-mortem. Most useful when built with `-g` and without `-O2` (debug build). For a core dump: `gdb ./my_node core`.

---

### System-level monitoring

```bash
# Real-time I/O stats — is disk write blocking a logger?
iostat -x 1

# Memory and swap pressure
vmstat 1

# Socket state — check DDS connections
ss -tunap | grep ros

# Check which process is holding a file/socket
lsof -p <pid>
lsof /dev/ttyUSB0    # who owns a serial port

# Check scheduling latency
sudo cyclictest -l 100000 -m -n -p80 -t1

# Monitor CPU frequency (detect thermal throttle)
watch -n1 cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq
```

---

### Common robotics debugging workflows

**"My control loop has intermittent latency spikes"**
1. `htop` → check if any core hits 100% at spike time
2. `cyclictest` → quantify worst-case scheduling jitter
3. `strace -c` → check for unexpected syscalls on the RT thread
4. `perf stat -e cache-misses` → cache miss rate during spikes
5. Check `mlockall` is called and `SCHED_FIFO` is set

**"ROS2 node keeps crashing"**
1. `gdb -p <pid>` then `bt` → get the stack trace
2. `valgrind --tool=memcheck` → heap corruption or use-after-free
3. `htop` → check if it's OOM-killed (D state before disappearing)

**"High CPU usage on a perception node"**
1. `perf record -g` then flamegraph → find the hot function
2. `htop` tree view → confirm it's one thread or all threads
3. Check if it's doing unnecessary allocations with `valgrind --tool=massif`

**"DDS topics dropping messages"**
1. `ss -tunap` → check socket buffers
2. `ros2 topic hz` → confirm publish rate
3. `ros2 doctor` → DDS diagnostics
4. Check QoS: BEST_EFFORT + unreliable network → expected drops

---

## System design patterns for interviews

### Pattern 1: multi-threaded robot control node

```
Thread                  Priority    Mechanism
─────────────────────────────────────────────
Control loop (1 kHz)    SCHED_FIFO 80   atomic read from sensor ring buf
Sensor reader           SCHED_FIFO 70   lock-free push to ring buf
State estimator         SCHED_FIFO 60   condition_variable trigger
Planner/LLM             SCHED_OTHER     separate process / lower priority
Logger                  SCHED_OTHER     async queue, buffered writes
```

Key points: pin control thread to a CPU core (`pthread_setaffinity_np`), lock memory (`mlockall`), pre-allocate all buffers.

### Pattern 2: design a surgical teleoperation system

1. **Input**: haptic device → high-priority thread, 1 kHz, SCHED_FIFO
2. **Motion scaling + tremor filter**: same thread, no alloc
3. **Arm command**: iceoryx zero-copy to control process (same host)
4. **Safety watchdog**: separate process, monitors heartbeat, trips e-stop on timeout
5. **Video**: separate process, lower priority, lossy is OK
6. **Logging**: async ring buffer → disk, never blocks control path

Latency budget breakdown → see [[Latency Budgets]].

### Pattern 3: warehouse autonomous mobile robot

1. **Perception**: camera + LIDAR → separate process, GPU inference
2. **State estimation**: EKF fusing wheel odometry + LIDAR scan matching
3. **Global planner**: A* or Dijkstra on costmap, runs on task arrival
4. **Local planner**: DWA/TEB at 20 Hz, avoids dynamic obstacles
5. **Controller**: velocity commands → motor driver at 100 Hz, SCHED_FIFO
6. **Fleet coordination**: REST/MQTT to central scheduler

### Common design interview traps

- **"Would you put the control loop in a Docker container?"** → Cautiously — containers add scheduling indirection. Validate jitter. For hard real-time, prefer host (or pin with cgroup). See [[Docker for ROS]].
- **"How do you handle sensor dropout?"** → Timeout in the estimator, fall back to dead reckoning, escalate to safe-stop after N missed frames.
- **"Where does the Kalman filter live?"** → Its own SCHED_FIFO thread at slightly lower priority than the actuator thread; publishes via lock-free queue or shared atomic struct.

---

## Where I've used it

- **Articulus**: multi-process ROS2 architecture — iceoryx for the hot control path, SCHED_FIFO for the control thread, `mlockall` to avoid page faults. Debugged a priority-inversion bug where the logging mutex was starving the control thread under load.
- **WALL-E V3**: diagnosed a timing jitter issue using `cyclictest` on the Orin Nano; found the USB camera driver interrupt was preempting the control thread. Fixed with CPU affinity separation.
- **FENCE-BOT**: used `strace` to find an unexpected `write()` syscall inside what should have been a lock-free path on the AVR bridge.

---

## Interview follow-ups

- **Q: Difference between a mutex and a semaphore?**
  - **A:** Mutex has ownership — only the thread that locks it can unlock it; the OS can boost its priority to prevent priority inversion. Semaphore has no ownership — any thread can signal (V) or wait (P) — useful for producer/consumer signaling. Use mutex for mutual exclusion of shared state; use semaphore (or condition_variable) for signaling.

- **Q: How do you prevent priority inversion in a real-time control loop?**
  - **A:** Use PI mutexes (`PTHREAD_PRIO_INHERIT`), which temporarily boost the lock-holder to the waiter's priority. Or avoid locks entirely on the hot path with lock-free ring buffers and atomics.

- **Q: What's a data race vs. a race condition?**
  - **A:** Data race: two threads access the same variable concurrently, at least one write, no synchronization — undefined behavior in C++. Race condition: a logic bug where the outcome depends on interleaving (can exist even with locks). All data races are UB; not all race conditions are data races.

- **Q: How would you design a 1 kHz control loop in a ROS2 node?**
  - **A:** Spawn a SCHED_FIFO thread (priority ~80), call `mlockall`, pre-allocate all buffers. Read sensor state from a lock-free ring buffer (written by a lower-priority sensor thread). Publish commands via iceoryx or a lock-free queue. Never call `malloc`, `new`, ROS2 logging (`printf` is OK), or anything that can block on that thread.

- **Q: What happens if you call `malloc` in a real-time thread?**
  - **A:** The allocator may `mmap` new pages (page fault, kernel call, non-deterministic latency). Or it may acquire an internal lock contended with another thread. Either breaks real-time guarantees. Pre-allocate everything before the RT loop starts.

- **Q: What is false sharing and how do you fix it?**
  - **A:** Two threads writing different variables that share a 64-byte cache line. Each write invalidates the other thread's cache entry, causing serial cache coherence traffic. Fix: `alignas(64)` to pad each hot variable to its own cache line.

---

## Gotchas / what trips me up

- Forgetting `mlockall` — the first page fault in the control thread shows up as a latency spike under load, not during testing.
- Using `std::cout` or ROS logging (`RCLCPP_INFO`) inside a real-time thread — both can malloc and lock.
- Confusing mutex ownership with semaphore non-ownership — semaphore doesn't protect against priority inversion by itself.
- Deadlock from inconsistent lock ordering — always lock in the same global order or use `std::lock(m1, m2)`.
- Forgetting to call `sched_setscheduler` as root — it silently fails without `CAP_SYS_NICE`.

---

## Links
- Related: [[Real-Time Determinism]], [[iceoryx Zero-Copy]], [[Docker for ROS]], [[Latency Budgets]], [[Safety-Critical Architecture]], [[Modern C++ for Robotics]], [[ROS2 Executors]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Process vs thread — key difference?
?
Processes have separate virtual address spaces (isolation: crash doesn't propagate). Threads share the same address space (cheaper to create, but a crash kills the whole process). ROS2 nodes = separate processes; callbacks = threads within a node.

Name four IPC mechanisms and when to pick each.
?
Pipe: simple parent↔child stream. Unix socket: fast bidirectional same-host. POSIX shared memory: fastest, zero-copy for large data (iceoryx). Message queue: structured msgs with priority between processes.

What is priority inversion and how do you fix it?
?
Low-priority thread holds a lock; high-priority thread blocks; medium-priority preempts the low one, starving the high-priority thread. Fix: PI mutex (PTHREAD_PRIO_INHERIT) boosts the lock-holder to the waiter's priority, or eliminate the lock with lock-free data structures.

Four conditions for deadlock?
?
Mutual exclusion, hold-and-wait, no preemption, circular wait. Break circular wait by always acquiring locks in the same global order.

Why can't you call malloc in a real-time thread?
?
malloc may mmap new pages (page fault → kernel trap, non-deterministic latency) or acquire an internal lock contended with another thread. Pre-allocate all memory before the RT loop.

What does mlockall do and why is it needed for real-time?
?
Locks all current and future virtual pages into physical RAM, preventing page faults during the RT loop. Without it, the first access to a new stack frame or buffer causes a kernel trap with non-deterministic latency.

What is false sharing?
?
Two threads write different variables that share a 64-byte cache line; each write invalidates the other's cache. Fix: pad variables with alignas(64) to put each on its own cache line.

Mutex vs semaphore — when to use each?
?
Mutex: mutual exclusion with ownership — only the locking thread unlocks it; OS can do priority inheritance. Semaphore: signaling — no ownership, any thread can V or P. Use mutex for shared state protection; use semaphore/condition_variable for producer-consumer signaling.

htop process states — what does D mean vs S?
?
S = sleeping (waiting for timer/I/O, normal). D = uninterruptible sleep (stuck in kernel I/O, bad — the process can't be killed). R = running. Z = zombie.

How do you find which function in a ROS2 node is consuming the most CPU?
?
`perf record -g` while the node runs, then `perf report` (or generate a flamegraph). Sort by self CPU% to find the hot function.

What does strace -c tell you and why is it useful for RT debugging?
?
Counts syscalls made by a process — type, count, time. Useful to find unexpected `futex` (lock contention), `mmap` (allocations), or `write` (logging) calls on a path that should be lock-free.

How do you pin a ROS2 node to a specific CPU core?
?
`taskset -c 2 ros2 run my_pkg my_node`. For a running process: `taskset -cp 2 <pid>`. Combined with `isolcpus=2` in kernel boot params to fully dedicate the core.

What is NUMA and when does it matter in robotics?
?
Non-Uniform Memory Access — on multi-socket or complex SoC systems, memory closer to a CPU node is faster (2-4×). On Jetson AGX or servers, bind control-loop process to NUMA node 0 with `numactl --cpunodebind=0 --membind=0` to avoid cross-node memory latency.

What is the fastest workflow to debug "my ROS2 node keeps crashing"?
?
(1) `gdb -p <pid>` + `bt` for stack trace. (2) `valgrind --tool=memcheck` for heap corruption. (3) `htop` to check if it's OOM-killed (D state before dying).
