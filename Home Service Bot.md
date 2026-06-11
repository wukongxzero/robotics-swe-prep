---

## type: concept domain: Perception & Navigation status: drafted last-reviewed: tags: [ros2, nav2, amcl, navigation, pick-and-place, mobile-robot]

# Home Service Bot

> [!question] Explain it cold
>
> - What does the home service bot do end-to-end?
> - How does AMCL localize the robot — what's the math?
> - Walk through the full autonomy stack from sensor to motor command

---

## Core idea

A home service robot that autonomously navigates an indoor environment, localizes itself on a known map, avoids obstacles, and performs pick-and-place tasks. Built on ROS2 with the Nav2 stack. Combines probabilistic localization (AMCL), sensor fusion (Kalman Filter), path planning, and manipulation.

---

## Full stack architecture

```
┌─────────────────────────────────────────────────────┐
│                 Mission Layer                        │
│         (Task: "go to kitchen, pick object")        │
└───────────────────┬─────────────────────────────────┘
                    │ goal pose
┌───────────────────▼─────────────────────────────────┐
│                Nav2 Stack                           │
│  BT Navigator → Global Planner → Local Planner      │
│                 (A*/Dijkstra)    (DWB/TEB/RPP)      │
└──────────┬──────────────────────────────────────────┘
           │ cmd_vel                │ costmaps
┌──────────▼──────────┐   ┌────────▼─────────────────┐
│   Motor Controllers  │   │   Costmap 2D             │
│   (diff drive)       │   │   (global + local)       │
└─────────────────────┘   └────────┬─────────────────┘
                                    │
┌───────────────────────────────────▼─────────────────┐
│              Sensor Layer                           │
│   LiDAR → AMCL (localization on map)               │
│   IMU + Encoders → Kalman Filter → /odom           │
│   Depth camera → obstacle detection                 │
└─────────────────────────────────────────────────────┘
```

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
   predicted LiDAR scan matches the actual scan against the map
   w_i = p(z | x_i, map)   (beam model or likelihood field)

4. Normalize weights: w_i = w_i / Σw_j

5. Resample: draw N particles with replacement weighted by w_i
   (particles cluster around likely poses)

6. Adaptive: shrink N when localized (low variance), grow when lost
```

**Key parameters:**
- `min_particles`, `max_particles` — N range (adaptive)
- `update_min_d`, `update_min_a` — minimum motion before update
- `laser_model_type` — `likelihood_field` (smooth, fast) vs `beam` (accurate)
- `recovery_alpha_slow/fast` — kidnapped robot recovery

**AMCL needs:**
- `/scan` (LiDAR)
- `/odom` (wheel odometry)
- A map (from `map_server` or SLAM output)
- TF tree: `map → odom → base_link`

---

## Kalman Filter for odometry

Fuses IMU (angular velocity, linear acceleration) with wheel encoders for clean `/odom`:

```
State: x = [x, y, θ, vx, vy, ω]ᵀ

Predict:
  x_k|k-1 = F·x_k-1 + B·u_k    (motion model from encoders)
  P_k|k-1 = F·P_k-1·Fᵀ + Q      (uncertainty grows)

Update (IMU measurement):
  y = z_k - H·x_k|k-1            (innovation)
  S = H·P_k|k-1·Hᵀ + R           (innovation covariance)
  K = P_k|k-1·Hᵀ·S⁻¹             (Kalman gain)
  x_k = x_k|k-1 + K·y            (corrected state)
  P_k = (I - K·H)·P_k|k-1        (corrected covariance)
```

In ROS2: `robot_localization` package handles this (`ekf_node` for extended KF).

---

## Nav2 stack components

| Component | What it does |
|-----------|-------------|
| `map_server` | Serves the pre-built occupancy grid map |
| `amcl` | Localizes robot on the map |
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

## Pick and place

```
Localization (AMCL) → navigate to object location
    ↓
Detect object (camera / depth sensor)
    ↓
Compute grasp pose (IK from [[Inverse Kinematics & DLS]])
    ↓
Arm motion planning (MoveIt2)
    ↓
Grasp execution (gripper)
    ↓
Navigate to drop location
    ↓
Place / release
```

**MoveIt2 in ROS2:**
```python
from moveit.planning import MoveItPy

moveit = MoveItPy(node_name="moveit_py")
arm = moveit.get_planning_component("arm")

arm.set_start_state_to_current_state()
arm.set_goal_state(pose_stamped_msg=target_pose, pose_link="end_effector")
plan = arm.plan()
if plan:
    moveit.execute(plan.trajectory, controllers=[])
```

---

## TF tree (critical to get right)

```
map
 └── odom          ← published by robot_localization (EKF)
      └── base_link  ← robot body frame
           ├── laser_frame   ← LiDAR
           ├── camera_frame  ← depth camera
           └── arm_base      ← manipulator base
```

**Rule:** `map → odom` published by AMCL (localization correction). `odom → base_link` published by odometry. Never mix them up — `odom` is continuous (no jumps), `map` has discontinuities when AMCL corrects.

---

## Common failure modes

| Problem | Cause | Fix |
|---------|-------|-----|
| Robot delocalized after moving | Particle filter diverged | Increase particles, check laser model quality |
| Nav2 stuck in recovery loop | Local costmap has phantom obstacle | `clear_costmap` service, check sensor noise |
| Arm can't find IK solution | Goal pose outside workspace | Check reachability, adjust target pose |
| TF extrapolation error | Clock sync issue or missing TF | Check `use_sim_time`, verify all nodes publishing TF |
| AMCL not converging | Wrong initial pose estimate | Use `initialpose` topic or set manually in RViz2 |

---

## Links

- Related: [[Kalman Filter]], [[Nav2 Stack]], [[SLAM & RTAB-Map]], [[Inverse Kinematics & DLS]], [[ROS2 Comm Patterns]], [[Sensor Fusion]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

How does AMCL localize a robot? ? Particle filter: scatter N particles (pose hypotheses) on the map, propagate with odometry (motion model + noise), weight each by how well its predicted LiDAR scan matches actual scan against map (sensor model), resample toward high-weight particles. Adapts N — more particles when uncertain, fewer when localized.

What is the TF tree structure for a Nav2 robot and what publishes each transform? ? map→odom: published by AMCL (localization corrections, can jump). odom→base_link: published by odometry/EKF (continuous, no jumps). base_link→sensors: static or published by robot_state_publisher from URDF.

What does the inflation layer in Nav2 costmap do? ? Inflates obstacle cells outward by the robot's inscribed radius + safety margin. This lets the planner treat the robot as a point and still guarantee clearance from obstacles. Too small → robot clips walls. Too large → narrow passages become impassable.
