---

## type: concept domain: SWE & math foundations status: drafted last-reviewed: 2026-06-02 tags: [cpp, real-time, swe, deep, differentiator]

# Modern C++ for Robotics

> [!star] The language your real-time stack lives in — RSE screens probe this Articulus + FENCE-BOT middleware were C++. "Real-time-safe C++" is a genuine differentiator — most candidates know C++ syntax; fewer know what's forbidden on a hot path and why. Own RAII, move semantics, and the no-alloc rule cold.

> [!question] Explain it cold
> 
> - What is RAII and why is it the foundation of safe C++?
> - What's forbidden on a real-time hot path and why?
> - Move vs copy — when does each happen and why care?

---

## Core idea

Modern C++ (11/14/17/20) gives you zero-overhead abstractions: high-level safety and expressiveness that compile down to code as fast as hand-written C. For robotics that means deterministic, real-time-capable code without manual memory bookkeeping — _if_ you know which features are real-time-safe and which silently aren't.

## Key facts

### RAII (Resource Acquisition Is Initialization)

- A resource's lifetime is bound to an object's scope: acquire in the constructor, release in the destructor. When the object goes out of scope, cleanup is automatic and **deterministic**.
- Why it's foundational: no manual `free`/`delete`, exception-safe (stack unwinding releases resources), no leaks. Smart pointers (`unique_ptr`, `shared_ptr`) are RAII for heap memory; `lock_guard` is RAII for mutexes.

**Without RAII — the problem:**
```cpp
fd_ = open("/dev/ttyUSB0", O_RDWR);
// ... 50 lines of code ...
if (some_error) return;   // fd_ never closed — resource leak
// ... more code ...
close(fd_);               // easy to forget, skipped on every error path
```

**With RAII — the fix:**
```cpp
// Destructor always runs — normal return, exception, early return, whatever
~MegaNode() {
    close(fd_);   // guaranteed cleanup, can't be forgotten
}
```

The key guarantee: **the destructor fires regardless of how the scope exits.** Exception thrown, early return, crash — doesn't matter. This is why exception-safety in C++ is built on RAII, not try/finally.

From your own code: `lock_guard<mutex>` unlocks on scope exit, `~MegaNode()` closes the serial port, `unique_ptr` deletes the heap object. You never call cleanup manually.

### Real-time-safe code (the differentiator)

- **No dynamic allocation on the hot path**: `new`/`malloc`/`std::vector` growth/`std::string` can take unbounded time and may block ([[Real-Time Determinism]]). Pre-allocate; reserve capacity up front; use fixed-size containers.
- **No unbounded blocking**: no lock contention you can't bound, no I/O, no `std::mutex` on the deadline path without care (priority inversion — use priority-inheritance).
- **No exceptions on the hot path** (often): exception unwinding is non-deterministic; many RT codebases disable or avoid them in the loop.
- **Avoid hidden allocations**: `std::function` can heap-allocate; some STL ops reallocate. Know what allocates.
- This is exactly the [[Real-Time Determinism|determinism]] discipline applied at the language level — and it's what made the Articulus control path bounded.

### Move semantics & ownership

- **Copy** duplicates; **move** transfers ownership (steals the guts, leaves the source empty) — cheap for things holding heap data. `std::move` casts to an rvalue to enable it.
- **Ownership model**: `unique_ptr` (single owner, move-only), `shared_ptr` (reference-counted, shared — atomic refcount has a cost), raw pointer/reference (non-owning observer). Express _who owns what_ in the type.
- Rule of zero/three/five: if you manage a resource manually you owe the special members; better, use RAII types so you owe none (rule of zero).

### Other modern essentials

- `const` correctness, `constexpr` (compile-time computation — zero runtime cost), templates/generics, `auto`, range-for, `std::optional`/`std::variant`, structured bindings.
- **Concurrency**: `std::thread`, `std::atomic`, lock-free patterns for the RT path; but real-time threading usually means OS priorities + pinning ([[Real-Time Determinism]]).

## Where I've used it

- **Articulus**: the real-time control stack in C++ — RAII for deterministic resource lifetime, no-alloc hot paths, careful ownership. The real-time-safe discipline above wasn't theory, it was the daily constraint.
- **FENCE-BOT**: the [[VR Teleop Pipeline|ROS2 C++ middleware]] (`vr_udp_publisher`, `robot_controller`) — modern C++ with [[colcon ament & launch|ament_cmake]].

## Interview follow-ups

- **Q:** What is RAII?
    - **A:** Bind a resource's lifetime to an object's scope — acquire in constructor, release in destructor — so cleanup is automatic, deterministic, and exception-safe. Smart pointers and lock_guard are RAII; it's why modern C++ doesn't need manual free.
- **Q:** What can't you do on a real-time control loop in C++?
    - **A:** No dynamic allocation (new/malloc/vector growth — unbounded time, may block), no unbounded locking or I/O, often no exceptions (non-deterministic unwinding), and watch hidden allocations (std::function, STL realloc). Pre-allocate and keep timing bounded.
- **Q:** Move vs copy?
    - **A:** Copy duplicates; move transfers ownership cheaply by stealing internals and leaving the source valid-but-empty. std::move enables it. Critical for performance with heap-owning types and for expressing single ownership via unique_ptr.
- **Q:** unique_ptr vs shared_ptr?
    - **A:** unique_ptr is a single move-only owner, zero overhead. shared_ptr is reference-counted shared ownership with an atomic refcount cost — use it only when ownership is genuinely shared, not by default.

## Gotchas / what trips me up

- Hidden heap allocations on the RT path (`std::function`, growing a `vector`, `std::string`).
- Reaching for `shared_ptr` by default — atomic refcounting isn't free; prefer `unique_ptr`.
- Forgetting `std::move` leaves the source in a valid-but-unspecified state (don't reuse it assuming old value).

## Links

- Related: [[Real-Time Determinism]], [[VR Teleop Pipeline]], [[colcon ament & launch]], [[DSA Patterns]], [[Git & Build Systems]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is RAII and why is it foundational? ? Bind a resource's lifetime to an object's scope (acquire in constructor, release in destructor) — automatic, deterministic, exception-safe cleanup. Smart pointers and lock_guard are RAII; it removes manual free/delete.

What's forbidden on a C++ real-time hot path, and why? ? Dynamic allocation (new/malloc/vector growth — unbounded time, may block), unbounded locking/I/O, often exceptions (non-deterministic unwinding), and hidden allocations (std::function, STL realloc). Pre-allocate; keep timing bounded.

Move vs copy in C++? ? Copy duplicates; move transfers ownership cheaply by stealing internals, leaving the source valid-but-empty (std::move enables it). Key for heap-owning types and single-ownership semantics.

unique_ptr vs shared_ptr — default choice? ? Prefer unique_ptr: single move-only owner, zero overhead. shared_ptr adds atomic reference-counting cost — use only for genuinely shared ownership.