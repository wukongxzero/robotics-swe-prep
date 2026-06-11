---

## type: project domain: Kinematics & Dynamics status: drafted last-reviewed: tags: [mujoco, ur10e, jacobian, kinematics, pd-control, python, nyu]

# Foundations of Robotics — UR10e Project

> [!question] Explain it cold
>
> - What are the four kinematics tasks in this project?
> - How does the PD controller work and what are its gains?
> - What is the relationship between the Jacobian and velocity FK/IK?

---

## What it is

NYU Foundations of Robotics final project. Implements four kinematics tasks on a **UR10e 6-DOF robot arm** in MuJoCo (from the MuJoCo Menagerie):
1. **Forward Kinematics — Position**: given joint angles → move arm to that configuration
2. **Inverse Kinematics — Position**: given a 3D target (red ball) → find joint angles to reach it
3. **Forward Kinematics — Velocity**: given joint velocities → predict EE velocity (Jacobian)
4. **Inverse Kinematics — Velocity**: given desired EE velocity → find joint velocities (Jacobian inverse)

All tasks use a PD controller to execute motion and MuJoCo for physics simulation. Output: animated GIFs showing the arm moving.

---

## UR10e model (MuJoCo Menagerie)

```python
model_path = "mujoco_menagerie/universal_robots_ur10e/scene.xml"
model = mujoco.MjModel.from_xml_path(model_path)
data  = mujoco.MjData(model)

# model.nq = 6    (6 joint angles)
# model.nu = 6    (6 actuators)
# model.opt.timestep = 0.002s (2ms physics step)

# Joint names: shoulder_pan, shoulder_lift, elbow,
#              wrist_1, wrist_2, wrist_3
# EE site: "attachment_site"
```

---

## PD Controller

```python
class PDController:
    def __init__(self, kp, kd, num_joints=6):
        self.kp = np.full(num_joints, kp)   # proportional gain
        self.kd = np.full(num_joints, kd)   # derivative gain
        self.target_angles = np.zeros(num_joints)

    def compute_control(self, current_pos, current_vel):
        error = self.target_angles - current_pos     # position error
        error_derivative = -current_vel               # velocity error (target vel = 0)
        torque = self.kp * error + self.kd * error_derivative
        return torque

# Used in project: kp=150, kd=20
controller = PDController(kp=150.0, kd=20.0, num_joints=6)
```

**Torque law:** `τ = Kp·(θ_d - θ) + Kd·(0 - θ̇)`

The derivative term (`-Kd·θ̇`) damps oscillation — without it, the arm overshoots and oscillates. `Kp=150` is the stiffness (how hard it pulls toward target), `Kd=20` is the damping.

---

## Velocity Controller (with gravity compensation)

For velocity tasks, a more sophisticated controller compensates for gravity:

```python
class VelocityController:
    def __init__(self):
        self.kd = np.array([800, 800, 800, 200, 200, 200])  # per-joint damping
        self.kp = np.array([500, 500, 500, 200, 200, 200])  # hold gains

    def compute_control(self, model, data):
        # Compute gravity + Coriolis forces via inverse dynamics
        data_copy.qpos[:] = data.qpos[:]
        data_copy.qvel[:] = data.qvel[:]
        data_copy.qacc[:] = 0
        mujoco.mj_inverse(model, data_copy)
        passive_forces = data_copy.qfrc_inverse[:6]   # gravity + Coriolis

        if moving:
            vel_error = target_velocities - current_vel
            torque = passive_forces + kd * vel_error
        else:
            # Hold position: PD on position error + gravity comp
            pos_error = initial_positions - current_pos
            torque = passive_forces + kp * pos_error + kd * (-current_vel)

        return torque
```

**Why `mj_inverse`?** Gravity and Coriolis forces change with joint configuration. Instead of hardcoding them, `mj_inverse` computes the exact forces needed to achieve zero acceleration — that's the feedforward term that cancels gravity. Without it, the arm sags under its own weight.

---

## Forward Kinematics — Position

```python
# Set random joint targets
thetas = [random angles...]
controller.set_all_targets(thetas)

# Run PD simulation — arm converges to target angles
frames, data = run_position_control_simulation(model_with_ball, controller, 5.0, 15, camera)
```

FK position is implicit here — MuJoCo's `mj_forward()` computes the EE position from joint angles automatically. The "answer" is visualized by where the arm ends up relative to the red ball.

---

## Inverse Kinematics — Position

```python
# Given: ball_xyz (random 3D position)
# Task: set joint angles so EE reaches ball

# IK is done manually — set joint angles as your solution
thetas = [-π/2, -π/2, -π/2, -π/2, -π/2, -π/2]
controller.set_all_targets(thetas)
```

The project uses a **numerical IK approach** — you set joint angles, run the simulation, and observe whether the EE reaches the ball. In practice, IK requires either:
- **Analytical IK** (UR10e has closed-form solution for 6-DOF)
- **Numerical IK** (Jacobian pseudoinverse, iterative)
- **DLS IK** (as used in FENCE-BOT — see [[Inverse Kinematics & DLS]])

---

## Forward Kinematics — Velocity (Jacobian)

```python
# Given: joint velocities q̇
# Task: predict EE velocity ẋ

# FK velocity: ẋ = J · q̇
# J is the 6×6 Jacobian (3 linear + 3 angular velocity rows)

# Get EE site position for arrow visualization
ee_site_id = mujoco.mj_name2id(model, mujoco.mjtObj.mjOBJ_SITE, "attachment_site")
ee_position = data.site_xpos[ee_site_id]

# Predict EE velocity (your guess visualized as cyan arrow)
predicted_ee_velocity = [0.3, 0.2, 0.1]  # must be computed via J·q̇
```

**The math:**
```
ẋ_ee = J(q) · q̇

where J is the geometric Jacobian:
  J = [J_v]   J_v: 3×n maps q̇ → linear EE velocity
      [J_w]   J_w: 3×n maps q̇ → angular EE velocity

For FK velocity, only first 3 joints were given non-zero velocity:
  joint_velocities = [ω₀, ω₁, ω₂, 0, 0, 0]
```

---

## Inverse Kinematics — Velocity (Jacobian inverse)

```python
# Given: desired EE velocity ẋ_d (random direction, 0.4 m/s)
# Task: find joint velocities q̇ such that J·q̇ = ẋ_d

# IK velocity: q̇ = J⁺ · ẋ_d
# J⁺ = pseudoinverse (J^T (J J^T)^{-1}) for overdetermined,
#      (J^T J)^{-1} J^T for underdetermined

# Your answer:
joint_velocities = [0.5, 0.3, -0.4, 0.0, 0.0, 0.0]
```

**The math:**
```
q̇ = J⁺ · ẋ_d

For a square J (6×6, full rank):  q̇ = J⁻¹ · ẋ_d
For rectangular J (3×6):          q̇ = Jᵀ(JJᵀ)⁻¹ · ẋ_d  (right pseudoinverse)
Near singularities:                q̇ = Jᵀ(JJᵀ + λ²I)⁻¹ · ẋ_d  (DLS — more stable)
```

---

## MuJoCo API patterns used

```python
# Load model and data
model = mujoco.MjModel.from_xml_path("scene.xml")
data  = mujoco.MjData(model)

# Reset simulation
mujoco.mj_resetData(model, data)

# Forward kinematics (compute positions from joint angles)
mujoco.mj_forward(model, data)

# Inverse dynamics (compute torques from motion state)
mujoco.mj_inverse(model, data)

# Step physics
data.ctrl[:] = torques
mujoco.mj_step(model, data)

# Read EE position via named site
site_id = mujoco.mj_name2id(model, mujoco.mjtObj.mjOBJ_SITE, "attachment_site")
ee_pos  = data.site_xpos[site_id]   # (3,) world-frame position

# Render frame
renderer = mujoco.Renderer(model, height=480, width=640)
renderer.update_scene(data, camera=camera)
frame = renderer.render()   # numpy uint8 array (H, W, 3)
```

---

## Key lessons

- **PD gains are coupled:** Kp=150 stiffness with Kd=20 damping works for this arm. Too high Kp with low Kd → oscillation. Too high Kd → overdamped, slow.
- **Gravity compensation is mandatory for velocity control:** Without `mj_inverse` feedforward, the arm sags and the velocity controller fights gravity instead of executing the motion.
- **The Jacobian connects joint space and task space:** FK velocity = J·q̇, IK velocity = J⁺·ẋ. Everything in velocity kinematics is a linear map through J.
- **MuJoCo sites vs bodies:** Use `data.site_xpos` for EE position (cleaner than computing through body chain). Sites are defined in the MJCF at fixed offsets from bodies.

---

## Links

- Related: [[Forward Kinematics]], [[Jacobian]], [[Inverse Kinematics & DLS]], [[MuJoCo & Gazebo]], [[PID Control]], [[FENCE-BOT]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is the PD torque law and what does each term do? ? τ = Kp·(θ_d - θ) + Kd·(0 - θ̇). Kp term: proportional restoring force toward target. Kd term: damping proportional to current velocity — prevents overshoot and oscillation. Without Kd, arm oscillates forever around the target.

What is the Jacobian velocity relationship (FK and IK)? ? FK velocity: ẋ = J·q̇ (joint velocities → EE velocity). IK velocity: q̇ = J⁺·ẋ (desired EE velocity → joint velocities). J is 6×n (3 linear + 3 angular rows). Near singularities, use DLS: q̇ = Jᵀ(JJᵀ + λ²I)⁻¹·ẋ.

Why use mj_inverse for gravity compensation in MuJoCo velocity control? ? mj_inverse computes the inverse dynamics — the exact torques needed to achieve a specified acceleration (zero here). Setting qacc=0 gives the gravity+Coriolis forces. Adding these as feedforward cancels gravity, so the PD gains only need to track the desired velocity, not fight the arm's weight.
