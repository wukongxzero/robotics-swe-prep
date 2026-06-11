---
type: concept
domain: ROS2 & middleware
status: drafted
last-reviewed: 2026-06-02
tags: [state-machine, FSM, HSM, behavior-trees, robotics, control]
---

# State Machines in Robotics


---

## Finite State Machine (FSM)

A set of **states**, **transitions** (triggered by events or conditions), and **actions** (executed on entry/exit/transition).

```
States:    MANUAL, AUTONOMOUS, IDLE
Events:    toggle, emergency_stop
Actions:   publish /cmd_vel, stop motors

MANUAL ──toggle──→ AUTONOMOUS ──toggle──→ MANUAL
MANUAL ──estop───→ IDLE
AUTONOMOUS ──estop→ IDLE
IDLE ──toggle────→ MANUAL  (always recover to MANUAL)
```

WALL-E's state_machine.cpp is a simple FSM. It's 3 states, clear transitions, easy to reason about.

---

## When FSMs break down

**State explosion:** N features × M modes = N×M states. A robot with 5 sensors and 4 operating modes needs 20 states — transitions become unmanageable.

**No history:** FSM has no memory of what happened before entering a state. A "retry" mechanism requires explicit retry states.

**Nested behavior:** representing "do A until B, then do C while monitoring D" is clumsy in flat FSM.

---

## Hierarchical State Machine (HSM / HFSM)

States can contain sub-states. A parent state handles events not handled by the active child. Reduces state explosion by factoring shared behavior into parent states.

```
OPERATIONAL (parent)
├── MANUAL
│     └── (handles joystick)
├── AUTONOMOUS
│     ├── NAVIGATING
│     └── RECOVERING
└── (OPERATIONAL handles: emergency_stop → IDLE for all children)

IDLE (peer of OPERATIONAL, not a child)
```

**History pseudo-state:** when re-entering a parent, resume the last active child state.

---

## FSM vs Behavior Tree vs GOAP

| | FSM | Behavior Tree | GOAP |
|--|-----|--------------|------|
| **Structure** | Graph of states | Tree of tasks | Goal + actions + planner |
| **Good at** | Mode switching, safety states | Complex task sequencing | Dynamic goal-directed behavior |
| **Bad at** | Many features (state explosion) | Low-level reactive control | Hard real-time constraints |
| **Used in** | WALL-E mode control, ROS2 lifecycle | Nav2 navigation tasks | Game AI, high-level robot planning |
| **Debuggability** | Very easy — finite states | Medium — tree traversal | Hard — emergent behavior |

**Rule:** use FSM for safety/mode control (few states, critical transitions). Use BT for task sequencing. Don't use FSM for complex multi-step tasks.

---

## ROS2 Lifecycle nodes — FSM built into ROS2

Every ROS2 lifecycle node IS a state machine:
```
Unconfigured ──configure──→ Inactive ──activate──→ Active
Active ──deactivate──→ Inactive ──cleanup──→ Unconfigured
Any state ──error──→ ErrorProcessing
```

Nav2 lifecycle_manager drives all Nav2 nodes through this FSM at startup.

---

## Implementation patterns

```cpp
// Simple enum FSM (WALL-E pattern)
enum class State { MANUAL, AUTONOMOUS, IDLE };
State state_ = State::MANUAL;

void toggle_callback(const std_msgs::msg::Bool::SharedPtr msg) {
    if (!msg->data) return;
    switch (state_) {
        case State::MANUAL:     state_ = State::AUTONOMOUS; break;
        case State::AUTONOMOUS: state_ = State::MANUAL;     break;
        case State::IDLE:       state_ = State::MANUAL;     break;
    }
}

// Better: state pattern (OOP)
// Each state is a class with enter(), update(), exit()
// Enables per-state behavior without giant switch statements
```

---

## Common FSM bugs in robotics

| Bug | Cause | Fix |
|-----|-------|-----|
| Stuck in state | Missing transition for an event | Audit all events × states — add explicit transitions |
| Wrong state after reconnect | No entry action on reconnect | Always define `on_enter()` actions |
| Concurrent state activation | Race condition on event | Mutex on state variable |
| Silent failure | No state logged | Publish state on every transition (WALL-E `/wall_e/state`) |

---

## Links
- Related: [[Behavior Trees]], [[ROS2 Node & Lifecycle]], [[WALL-E V3]], [[Safety-Critical Architecture]]
- Parent: [[00 Knowledge Map]]

---

