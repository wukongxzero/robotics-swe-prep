---
type: concept
domain: SWE & math foundations
status: drafted
last-reviewed:
tags: [design-patterns, cpp, swe, interview-prep]
---

# Design Patterns

> [!question] Explain it cold
> *Answer from memory first. 30 min nightly until drilled.*
>
> - Name the three categories of design patterns.
> - For each pattern below: what problem does it solve, and sketch the structure.
> - Which patterns show up most in robotics codebases?

---

## Core idea

Design patterns are **named solutions to recurring structural problems** in OOP code. They aren't code to copy — they're a vocabulary. An interviewer who asks "how would you structure this?" is often fishing for a pattern name + the reasoning behind it.

---

## The patterns to own cold

### Creational — *how objects are created*

**Singleton**
- Problem: exactly one instance of a class exists (e.g., a ROS2 node manager, a hardware driver).
- Structure: private constructor, static `getInstance()` that creates on first call and returns the same pointer every time.
- Gotcha: not thread-safe without a mutex or `std::call_once`. In robotics, hardware driver singletons are common.

**Factory Method**
- Problem: create objects without specifying the exact class at compile time (e.g., spawn different controller types based on config).
- Structure: base class declares a `create()` virtual method; subclasses override it to return their specific type.
- Robotics use: plugin systems — `ControllerFactory::create("LQR")` returns an `LQRController`.

**Abstract Factory**
- Problem: create families of related objects (e.g., a full sensor suite — IMU + encoder + camera — for sim vs hardware).
- Structure: an interface with methods for each product type; concrete factories implement the whole family.

---

### Structural — *how objects are composed*

**Adapter**
- Problem: make an incompatible interface work with existing code (e.g., wrap a vendor SDK with your internal sensor interface).
- Structure: adapter class holds a reference to the adaptee and translates method calls.
- Robotics use: wrapping a Dynamixel SDK call behind a generic `Motor::setVelocity()` interface.

**Decorator**
- Problem: add behavior to an object at runtime without subclassing (e.g., add logging or rate-limiting to a controller).
- Structure: decorator wraps the original object and implements the same interface, adding behavior before/after delegation.

**Facade**
- Problem: provide a simple interface to a complex subsystem (e.g., a `RobotArm` class that hides FK, IK, and joint commands behind `moveTo(pose)`).
- Structure: a single class that delegates to multiple subsystem classes.
- Robotics use: common — hides the complexity of a multi-node pipeline behind one interface.

**Composite**
- Problem: treat individual objects and groups of objects uniformly (e.g., a behavior tree where a leaf action and a sequence node are both "nodes").
- Structure: Component interface, Leaf implements it, Composite holds children and delegates.
- Robotics use: **behavior trees** in Nav2 are literally this pattern.

---

### Behavioral — *how objects communicate*

**Observer**
- Problem: notify multiple objects when one object's state changes (e.g., multiple subscribers to a sensor topic).
- Structure: Subject holds a list of Observer pointers; calls `notify()` on all of them when state changes.
- Robotics use: ROS2 pub/sub is Observer pattern over DDS. Also used for GUI dashboards watching robot state.

**Strategy**
- Problem: swap algorithms at runtime (e.g., switch between PID and LQR without changing the control loop code).
- Structure: Context holds a pointer to a Strategy interface; call `strategy->compute(state)`. Swap the pointer to change behavior.
- Robotics use: controller switching — `setStrategy(new LQRController())`.

**Command**
- Problem: encapsulate a request as an object — enables queuing, undo, logging (e.g., a teleoperation command queue).
- Structure: Command interface with `execute()` and optionally `undo()`. Invoker holds and calls commands.
- Robotics use: motion command queuing in teleop pipelines; action servers.

**State**
- Problem: an object's behavior depends on its state, and it transitions between states (e.g., a robot in MANUAL/AUTONOMOUS/IDLE).
- Structure: Context delegates to a State object; each state implements the same interface differently. State transitions swap the State pointer.
- Robotics use: **heavily used** — WALL-E V3 state machine is this pattern. Every robot lifecycle is a state machine.

**Template Method**
- Problem: define the skeleton of an algorithm in a base class, let subclasses fill in the steps.
- Structure: base class has `run()` calling `step1()`, `step2()` etc. — some implemented, some pure virtual.
- Robotics use: controller base classes — `update()` calls `computeError()` then `computeOutput()`, subclasses override the steps.

---

## Robotics-specific frequency

Most common in robotics interviews and codebases:
1. **State** — every robot is a state machine
2. **Observer** — pub/sub everywhere
3. **Strategy** — controller/planner swapping
4. **Facade** — hiding subsystem complexity
5. **Composite** — behavior trees (Nav2)
6. **Singleton** — hardware drivers

## Interview follow-ups
- **Q:** What's the difference between Strategy and State?
  - **A:** Strategy swaps an algorithm from outside (the context doesn't know which it has); State manages its own transitions internally and behavior changes as the state changes. Strategy is "plug in different implementations"; State is "I am in a mode and my behavior follows."
- **Q:** How is Observer different from pub/sub?
  - **A:** They're the same pattern at different scales — Observer is the OOP pattern (direct callback registration), pub/sub is Observer distributed over a message bus (like ROS2 topics or DDS).
- **Q:** When would you use Decorator over subclassing?
  - **A:** When you want to mix and match behaviors at runtime without a class explosion. Three features combinable means 8 subclasses vs 3 decorators.

## Gotchas / what trips me up
- Singleton thread-safety: double-checked locking is broken in C++ without `std::call_once` or `static` local variable (Meyers Singleton — the `static` inside a function IS thread-safe in C++11+).
- State vs Strategy confusion under pressure — remember: State manages its own transitions, Strategy doesn't.

## Links
- Related: [[Modern C++ for Robotics]], [[Safety-Critical Architecture]], [[Nav2 Stack]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Name the three categories of design patterns.
?
Creational (how objects are created), Structural (how objects are composed), Behavioral (how objects communicate).

What problem does the Strategy pattern solve?
?
Swap algorithms at runtime without changing the calling code. Context holds a Strategy pointer; swap the pointer to change behavior.

What problem does the State pattern solve?
?
An object whose behavior depends on which state it's in. Each state implements the same interface; transitions swap the state object.

What's the difference between Strategy and State?
?
Strategy is swapped from outside and doesn't manage transitions. State manages its own transitions and changes behavior as the object moves through modes.

What design pattern do behavior trees (Nav2) implement?
?
Composite — leaf nodes and composite nodes both implement the same Node interface; the tree is built by nesting them.

What is the Singleton gotcha in C++?
?
Thread safety. Use Meyers Singleton (static local variable) or std::call_once. Double-checked locking without atomics is broken.

What is the Observer pattern, and where does it appear in ROS2?
?
Subject notifies a list of Observers when state changes. ROS2 pub/sub is Observer over DDS — subscribers are observers of a topic.
