---
type: concept
domain: Perception & ML
status: drafted
last-reviewed: 2026-06-02
tags: [behavior-trees, nav2, task-planning, robotics, BT]
---

# Behavior Trees

> [!question] Explain it cold
> - What is a behavior tree and how does it differ from a state machine?
> - What are the four node types in a BT?
> - How does Nav2 use behavior trees for navigation?

---

## Core idea

A behavior tree is a hierarchical task structure where a **root node** ticks child nodes every cycle. Each node returns **Success**, **Failure**, or **Running**. The tree structure (Sequence, Fallback, Parallel) determines how results propagate. More modular and reusable than FSMs — adding new behaviors doesn't require modifying existing transitions.

---

## Four node types

### Leaf nodes (do the actual work)

**Action node:** executes something, returns Running until done, then Success or Failure.
```
NavigateToPose → Running (robot moving) → Success (arrived) / Failure (aborted)
```

**Condition node:** checks something, returns Success or Failure instantly (no Running).
```
IsGoalReached? → Success / Failure
IsBatteryLow?  → Success / Failure
```

### Composite nodes (control flow)

**Sequence (→):** ticks children left-to-right. Returns Success only if ALL children succeed. Returns Failure on first failure (like AND).
```
Sequence: [ComputePath → FollowPath] 
→ both must succeed for navigation to complete
```

**Fallback / Selector (?):** ticks children left-to-right. Returns Success on first success. Returns Failure only if ALL fail (like OR / try-next-option).
```
Fallback: [NavigateToPose → RecoverFromFailure → Abort]
→ try navigation, if it fails try recovery, if that fails abort
```

**Parallel:** ticks ALL children simultaneously. Returns Success if N of M children succeed (configurable threshold).

---

## BT vs FSM — key differences

| | FSM | Behavior Tree |
|--|-----|--------------|
| **Structure** | Graph (states + transitions) | Tree (hierarchical tasks) |
| **Adding behavior** | Add states + transitions (can break existing) | Add subtree (modular) |
| **Reactivity** | Event-driven transitions | Every tick re-evaluates conditions |
| **Complexity** | Explodes with features (state explosion) | Scales via hierarchy |
| **Debugging** | Easy (finite states) | Harder (traversal order) |
| **Best for** | Safety-critical mode control | Complex task sequencing |

---

## Nav2's behavior tree

Nav2's `bt_navigator` uses XML behavior trees to define navigation behavior.

```xml
<!-- navigate_to_pose_w_replanning_and_recovery.xml (simplified) -->
<root>
  <BehaviorTree>
    <Sequence>
      <RateController hz="1.0">
        <RecoveryNode number_of_retries="6">
          <Sequence>
            <ComputePathToPose goal="{goal}"/>
            <FollowPath path="{path}"/>
          </Sequence>
          <Sequence>
            <ClearEntireCostmap service_name="local_costmap/clear_entirely_local_costmap"/>
            <Spin spin_dist="1.57"/>
            <BackUp backup_dist="0.3"/>
          </Sequence>
        </RecoveryNode>
      </RateController>
    </Sequence>
  </BehaviorTree>
</root>
```

**Reading it:**
1. Every 1Hz: try [ComputePath → FollowPath]
2. If that fails (up to 6 retries): clear costmap → spin → back up
3. Each retry re-plans the path

---

## BehaviorTree.CPP (the library Nav2 uses)

```cpp
#include <behaviortree_cpp/bt_factory.h>

// Custom action node
class MyAction : public BT::StatefulActionNode {
public:
    MyAction(const std::string& name, const BT::NodeConfig& config)
        : BT::StatefulActionNode(name, config) {}

    BT::NodeStatus onStart() override {
        // start the action
        return BT::NodeStatus::RUNNING;
    }

    BT::NodeStatus onRunning() override {
        if (done_) return BT::NodeStatus::SUCCESS;
        return BT::NodeStatus::RUNNING;
    }

    void onHalted() override { /* cleanup */ }
};

// Register and run
BT::BehaviorTreeFactory factory;
factory.registerNodeType<MyAction>("MyAction");
auto tree = factory.createTreeFromFile("my_tree.xml");
tree.tickWhileRunning();
```

---

## Common interview questions

**Q: How is BT different from FSM?**
A: BTs are hierarchical and re-evaluate conditions every tick — more modular and reactive. FSMs are graph-based with explicit transitions — simpler but suffer state explosion. Use FSM for safety-critical mode control, BT for complex task sequencing.

**Q: What happens when FollowPath fails in Nav2?**
A: The Sequence returns Failure. The RecoveryNode catches it and runs the recovery subtree (clear costmap, spin, back up). After recovery, it retries the navigation Sequence up to N times.

**Q: What does "ticking" mean in a BT?**
A: The tree is traversed from root every control cycle. Each node executes its logic and returns Success/Failure/Running. Running propagates up — the tree remembers where it was and resumes on the next tick.

---

## Links
- Related: [[Nav2 Stack]], [[State Machines in Robotics]], [[Motion Planning Algorithms]], [[Safety-Critical Architecture]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Four BT node types and what each returns? ? Action: executes work, returns Running/Success/Failure. Condition: checks instantly, returns Success/Failure only. Sequence: all children must succeed (AND logic). Fallback: first success wins (OR logic / try alternatives).

Sequence vs Fallback node — behavior? ? Sequence: ticks children left-to-right, returns Success only if ALL succeed, Failure on first failure. Fallback: ticks children left-to-right, returns Success on first success, Failure only if ALL fail. Sequence = AND. Fallback = OR / try-next.

How does Nav2 use behavior trees? ? bt_navigator loads an XML BT that orchestrates: compute path → follow path → recovery on failure (spin, back up, clear costmap). The BT re-ticks every cycle, enabling reactive replanning when the path fails or obstacles appear.
