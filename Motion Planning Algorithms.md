---
type: concept
domain: Perception & ML
status: drafted
last-reviewed: 2026-06-02
tags: [motion-planning, path-planning, A-star, RRT, PRM, configuration-space, nav2]
---

# Motion Planning Algorithms

> [!question] Explain it cold
> - What is configuration space and why do planners work in it?
> - Explain A* — how does it differ from Dijkstra?
> - When do you use RRT vs PRM vs A*?

---

## Configuration space (C-space)

A robot with N joints has N degrees of freedom. C-space maps every possible joint configuration to a single point in N-dimensional space. Obstacles in workspace become obstacle *regions* in C-space.

**Why:** planning in C-space reduces any robot to a point-to-point planning problem. You don't plan the full robot shape — you plan a path for the configuration point.

```
2-DOF arm: C-space is 2D (θ1, θ2)
Mobile robot: C-space is 3D (x, y, θ)
Collision checking: is this C-space point inside an obstacle region?
```

---

## Graph-based planners (complete, optimal)

### Dijkstra
- Expand nodes by cost from start. Priority queue ordered by g(n) (cost so far).
- Guarantees shortest path. Explores in all directions — slow for large spaces.
- O((V + E) log V)

### A*
- Adds heuristic h(n) (estimated cost to goal). Priority queue ordered by f(n) = g(n) + h(n).
- Heuristic guides search toward goal — much faster than Dijkstra.
- **Admissible heuristic:** h(n) ≤ true cost. Guarantees optimal path.
- Common heuristics: Euclidean distance, Manhattan distance.

```python
# A* key insight: f = g + h
# g = cost from start to current node
# h = estimated cost from current to goal (must be admissible)
# f = total estimated path cost through this node

open_set = PriorityQueue()  # ordered by f
open_set.put((0, start))

while open_set:
    f, current = open_set.get()
    if current == goal: return path
    for neighbor in graph[current]:
        g = g_scores[current] + cost(current, neighbor)
        if g < g_scores[neighbor]:
            g_scores[neighbor] = g
            f = g + heuristic(neighbor, goal)
            open_set.put((f, neighbor))
```

**In Nav2:** global planner uses NavFn (Dijkstra/A* on occupancy grid) or Smac (A* with kinematic constraints).

---

## Sampling-based planners (probabilistically complete)

For high-dimensional C-spaces where grid-based search is intractable.

### RRT (Rapidly-exploring Random Tree)
- Grow a tree from start by sampling random configs and extending toward them.
- Fast for high-DOF, finds a path quickly, not optimal.

```
1. Sample random config q_rand
2. Find nearest node q_near in tree
3. Extend from q_near toward q_rand by step size δ → q_new
4. Collision check q_new
5. If valid: add to tree
6. Repeat until q_new ≈ goal
```

### RRT*
- Same as RRT but rewires the tree to minimize cost.
- Asymptotically optimal — path improves as more samples are added.
- Slower than RRT but finds better paths.

### PRM (Probabilistic Roadmap)
- **Two phases:** (1) build roadmap offline by sampling valid configs and connecting nearby ones. (2) query: connect start/goal to roadmap, run A*.
- Best when: many queries in same environment (pre-compute once, reuse).
- Bad for dynamic environments (roadmap becomes stale).

---

## Comparison

| Algorithm | Complete | Optimal | Speed | Best for |
|-----------|---------|---------|-------|----------|
| Dijkstra | Yes | Yes | Slow | Small graphs, guaranteed shortest path |
| A* | Yes | Yes (admissible h) | Fast | 2D grid maps (Nav2 global planner) |
| RRT | Probabilistic | No | Fast | High-DOF, quick path finding |
| RRT* | Probabilistic | Asymptotic | Medium | High-DOF, better paths over time |
| PRM | Probabilistic | No | Fast (query) | Static environments, many queries |

---

## Local planners (reactive, for controller server)

Global planner finds a path. Local planner follows it while avoiding dynamic obstacles.

**DWB (Dynamic Window Approach):** samples velocity commands in the robot's dynamic window (reachable velocities given current velocity + acceleration limits). Scores trajectories by path alignment, goal distance, obstacle clearance. Used in WALL-E's Nav2 config.

**TEB (Timed Elastic Band):** optimizes a trajectory as a rubber band between start and goal, subject to kinematic and obstacle constraints.

---

## Where these appear in Nav2 (WALL-E)

```yaml
# nav2_params.yaml
planner_server:
  GridBased:
    plugin: "nav2_navfn_planner::NavfnPlanner"  # A* / Dijkstra on occupancy grid

controller_server:
  FollowPath:
    plugin: "dwb_core::DWBLocalPlanner"  # Dynamic Window Approach
```

---

## Links
- Related: [[Nav2 Stack]], [[Behavior Trees]], [[Trajectory Optimization]], [[acados OCS2 CasADi]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

A* vs Dijkstra — one key difference? ? A* adds a heuristic h(n) (estimated cost to goal) to the priority: f = g + h. Dijkstra only uses g (cost from start). The heuristic guides A* toward the goal, making it much faster. Both guarantee optimal paths when the heuristic is admissible (never overestimates).

When do you use RRT vs PRM? ? RRT: single query in unknown/dynamic environment — fast path finding, not optimal. PRM: many queries in the same static environment — build roadmap once offline, query fast repeatedly. If the environment changes, PRM becomes stale.

What is configuration space and why do planners use it? ? C-space maps every robot configuration (joint angles or pose) to a single point in N-dimensional space. Obstacles become obstacle regions. Planning reduces to point-to-point path finding regardless of robot complexity. A 6-DOF arm plans in 6D C-space; a mobile robot plans in 3D (x,y,θ).

What is an admissible heuristic and why does it matter for A*? ? Admissible = never overestimates the true cost to goal. h(n) ≤ true_cost(n, goal). Guarantees A* finds the optimal path. Euclidean distance is admissible for uniform-cost grids. If h overestimates, A* may miss the optimal path.
