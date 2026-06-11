---

## type: concept domain: Perception & ML status: drafted last-reviewed: tags: [perception, nav2, navigation, planning, amcl, ros2]

# Nav2 Stack


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


---


## Common failure modes

| Problem | Cause | Fix |
|---------|-------|-----|
| Robot delocalized after moving | Particle filter diverged | Increase `max_particles`, check laser model quality |
| Nav2 stuck in recovery loop | Local costmap has phantom obstacle | `clear_costmap` service, check sensor noise |
| AMCL not converging | Wrong initial pose estimate | Publish to `/initialpose` or set manually in RViz2 |
| TF extrapolation error | Clock sync issue or missing TF | Check `use_sim_time`, verify all TF publishers running |
| `NavigateToPose` fails silently | Goal outside map or in obstacle | Check costmap values at goal, verify reachability |


## Links

- Related: [[Home Service Bot]], [[SLAM & RTAB-Map]], [[ROS2 Comm Patterns]], [[YOLOv8 Detection]], [[Kalman Filter]], [[MPC & Virtual Fixtures]]
- Parent: [[00 Knowledge Map]]

---

