---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [perception, nav2, navigation, planning, amcl, ros2]

# Nav2 Stack

> [!question] Explain it cold
> 
> - What does Nav2 do, and what are its major components?
> - Global vs local planner — what's the split?
> - What's a costmap, and why two of them?
> - How does AMCL localize the robot?

---

## Core idea

Nav2 is ROS2's navigation framework — given a map and a goal pose, it plans and executes collision-free motion to get there. It's architected as a set of servers coordinated by **behavior trees**, which makes the navigation logic configurable and recoverable rather than a monolithic state machine.

---

## Key facts & formulas

- **Costmaps** (2D occupancy grids annotated with cost):
    - **Global costmap** — whole known map, used for long-range planning.
    - **Local costmap** — small rolling window around the robot, updated from live sensors, used for immediate obstacle avoidance.
    - Cost layers: static (the map), obstacle (live sensor hits), inflation (buffer around obstacles so the planner respects robot radius).
- **Planners**:
    - **Global planner** — computes a full path to the goal over the global costmap (A*, Dijkstra, Theta*, or the Smac planners). Runs less often.
    - **Local planner / controller** — follows the global path while reacting to live obstacles (DWB, TEB, MPPI) — produces actual velocity commands at control rate.
- **Behavior Trees**: Nav2 orchestrates navigation as a BT — ticking nodes for "compute path," "follow path," "recovery" (spin, back up, clear costmap). Recoverable and composable vs a flat state machine.
- **Exposed as ROS2 [[ROS2 Comm Patterns|actions]]**: `NavigateToPose` is an action — long-running, feedback, cancellable. (Ties back to why actions exist.)
- **AMCL** for localization on a known map (particle filter), or pose from [[SLAM & RTAB-Map|SLAM]] when mapping.

---

## Nav2 component table

| Component | What it does |
|-----------|-------------|
| `map_server` | Serves the pre-built occupancy grid map |
| `amcl` | Localizes robot on the map (particle filter) |
| `nav2_costmap_2d` | Builds global + local costmaps from LiDAR + map |
| `nav2_planner` (A*) | Global path from current pose to goal |
| `nav2_controller` (DWB/RPP) | Local trajectory following, obstacle avoidance |
| `nav2_bt_navigator` | Behavior Tree orchestrating the full nav pipeline |
| `nav2_recoveries` | Spin, back up, clear costmap when stuck |
| `lifecycle_manager` | Manages node startup/shutdown order |

**Costmap layers:**
- `static_layer` — known obstacles from map
- `obstacle_layer` — real-time LiDAR hits
- `inflation_layer` — inflates obstacles by robot radius + safety margin

---

## AMCL — Adaptive Monte Carlo Localization

**What it does:** Estimates robot pose `(x, y, θ)` on a known 2D map using a particle filter. Doesn't build the map — uses a pre-built one (from SLAM).

**Algorithm:**
```
1. Initialize: scatter N particles uniformly (or near known start pose)
   particles = [(x_i, y_i, θ_i, w_i) for i in range(N)]

2. Predict (motion model): for each particle, apply odometry + noise
   x_i += Δx + noise_x
   y_i += Δy + noise_y
   θ_i += Δθ + noise_θ

3. Update (sensor model): weight each particle by how well its
   predicted LiDAR scan matches actual scan against the map
   w_i = p(z | x_i, map)   (beam model or likelihood field)

4. Normalize weights: w_i = w_i / Σw_j

5. Resample: draw N particles with replacement weighted by w_i
   (particles cluster around likely poses)

6. Adaptive: shrink N when localized (low variance), grow when lost
```

**Key parameters:**
- `min_particles`, `max_particles` — N range (adaptive)
- `update_min_d`, `update_min_a` — minimum motion before update runs
- `laser_model_type` — `likelihood_field` (smooth, fast) vs `beam` (accurate)
- `recovery_alpha_slow/fast` — kidnapped robot recovery (inject random particles when confidence drops)

**AMCL needs:**
- `/scan` (LiDAR)
- `/odom` (wheel odometry)
- A map (from `map_server` or SLAM output)
- TF tree: `map → odom → base_link`

---

## TF tree (critical)

```
map
 └── odom          ← published by AMCL (localization correction)
      └── base_link  ← robot body frame
           ├── laser_frame   ← LiDAR
           ├── camera_frame  ← depth camera
           └── arm_base      ← manipulator base
```

**Rule:** `map → odom` published by AMCL — has discontinuities when AMCL corrects pose. `odom → base_link` published by odometry/EKF — continuous, no jumps. Never mix them up. Confusing the two causes delocalization or TF extrapolation errors.

---

## KF odometry fusion

`robot_localization` package (`ekf_node`) fuses IMU + encoders for clean `/odom`:
```
State: [x, y, θ, vx, vy, ω]ᵀ
Predict: motion model from encoders (uncertainty grows via Q)
Update:  IMU measurement corrects with Kalman gain
```
See [[Kalman Filter]] for full math.

---

## Where I've used it

- **Home Service Bot** ([[Home Service Bot]]): full Nav2 stack — AMCL localization, A* global planner, DWB local controller, costmap 2D, pick-and-place pipeline with MoveIt2. AMCL particle filter drove localization from LiDAR on a pre-built map.
- **WALL-E V3 sim (2026-06-04 validated)**: Full Nav2 + RTAB-Map stack in Gazebo Harmonic. Robot drives to NavigateToPose goals autonomously. Key lessons:
  - `controller_server` remaps `cmd_vel` to an intermediate topic for the velocity smoother — if smoother remappings are wrong, nothing reaches `/cmd_vel` and the robot sits still despite goal being accepted. Verify the full chain: controller → smoother → `/cmd_vel` → diff drive
  - Error code 204 = controller (DWB) failure. Can happen because: map is empty (no obstacles = DWB confused), cmd_vel not reaching diff drive, or TF too stale
  - `transform_tolerance` defaults to 0.3s. With RTAB-Map at 1Hz, the map→odom TF is up to 1s old between updates — set `transform_tolerance: 1.5` in both local and global costmap params
  - An empty map (RTAB-Map built nothing) still allows the planner to accept a goal but DWB may abort immediately

---

## Interview follow-ups

- **Q:** Global vs local planner?
    - **A:** Global plans a full path to the goal over the global costmap (A*/Dijkstra-style), runs occasionally. Local/controller follows that path at control rate while avoiding live obstacles (DWB/TEB/MPPI), emitting velocity commands.
- **Q:** Why two costmaps?
    - **A:** Global covers the whole map for long-range planning; local is a small sensor-updated rolling window for immediate obstacle avoidance. Different scope, different update rate.
- **Q:** Why behavior trees instead of a state machine?
    - **A:** BTs make navigation logic composable and recoverable — recovery behaviors (clear costmap, spin, back up) slot in cleanly, and the tree is reconfigurable without rewriting control flow.
- **Q:** What's costmap inflation?
    - **A:** A cost buffer expanded around obstacles by the robot's radius so the planner keeps a safe margin and treats the robot as a point.
- **Q:** How does AMCL differ from SLAM?
    - **A:** AMCL localizes on a known map (particle filter, no map building). SLAM builds the map while localizing simultaneously. AMCL needs a pre-built map; SLAM creates one.

## Common failure modes

| Problem | Cause | Fix |
|---------|-------|-----|
| Robot delocalized after moving | Particle filter diverged | Increase `max_particles`, check laser model quality |
| Nav2 stuck in recovery loop | Local costmap has phantom obstacle | `clear_costmap` service, check sensor noise |
| AMCL not converging | Wrong initial pose estimate | Publish to `/initialpose` or set manually in RViz2 |
| TF extrapolation error | Clock sync issue or missing TF | Check `use_sim_time`, verify all TF publishers running |
| `NavigateToPose` fails silently | Goal outside map or in obstacle | Check costmap values at goal, verify reachability |

## Gotchas / what trips me up

- Confusing AMCL (localize on known map) with SLAM (build the map).
- Forgetting `NavigateToPose` is an action (cancellable/long-running), not a service.
- `map → odom` can jump (AMCL corrections); `odom → base_link` never does — mixing them up breaks sensor fusion.
- Particle filter needs motion to converge — robot must move for weights to differentiate pose hypotheses.

## Links

- Related: [[Home Service Bot]], [[SLAM & RTAB-Map]], [[ROS2 Comm Patterns]], [[YOLOv8 Detection]], [[Kalman Filter]], [[MPC & Virtual Fixtures]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Global vs local planner in Nav2? ? Global: full path to goal over the global costmap (A*/Dijkstra/Smac), runs occasionally. Local/controller: follows the path at control rate while avoiding live obstacles (DWB/TEB/MPPI), emits velocity commands.

Why does Nav2 use two costmaps? ? Global costmap = whole map for long-range planning; local costmap = small sensor-updated rolling window for immediate obstacle avoidance. Different scope and update rate.

Why behavior trees over a flat state machine in Nav2? ? BTs make navigation composable and recoverable — recovery behaviors (clear costmap, spin, back up) slot in cleanly and the logic reconfigures without rewriting control flow.

What is costmap inflation? ? A cost buffer grown around obstacles by the robot's radius, so the planner keeps a safe margin and can treat the robot as a point. Too small → clips walls. Too large → narrow passages become impassable.

How does AMCL localize a robot? ? Particle filter: scatter N particles (pose hypotheses), propagate with odometry + noise (motion model), weight each by how well its predicted LiDAR scan matches actual scan against map (sensor model), resample toward high-weight particles. Adaptive N — more when uncertain, fewer when localized.

What is the TF tree for Nav2 and what publishes each link? ? map→odom: AMCL (can jump on corrections). odom→base_link: odometry/EKF (continuous, no jumps). base_link→sensors: robot_state_publisher from URDF. Never confuse map and odom — odom is continuous, map has pose corrections.
