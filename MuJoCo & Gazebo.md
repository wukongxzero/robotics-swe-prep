---

## type: concept domain: Simulation & tooling status: drafted last-reviewed: tags: [sim, mujoco, gazebo, contact, python, ur10e]

# MuJoCo & Gazebo

> [!question] Explain it cold
> 
> - What is MuJoCo known for, and why control researchers love it?
> - What's Gazebo's niche?
> - How do you pick a simulator for a given robotics problem?
> - Walk through the MuJoCo API: load → step → read EE position

---

## Core idea

Different simulators optimize for different things. **MuJoCo** (Multi-Joint dynamics with Contact) is prized for fast, stable, accurate **contact dynamics** and is the default for control/RL research. **Gazebo** is the classic ROS-integrated simulator — good sensor models and ecosystem fit. **Isaac** ([[Isaac Lab]]) brings GPU parallelism and rendering. Picking one is about matching the simulator's strength to your problem.

## Key facts & formulas

- **MuJoCo**:
    - Soft-constraint contact model — approximates rigid contact with a smooth, well-conditioned formulation, so contact-rich sims stay fast and stable (no blowups).
    - Analytical derivatives / differentiable-friendly — good for gradient-based control and trajectory optimization.
    - Now open-source (DeepMind); the go-to for locomotion/manipulation RL and control research.
- **Gazebo** (Classic and the newer Gz/Ignition):
    - Tight ROS/ROS2 integration, rich sensor plugins (lidar, camera, IMU), URDF/SDF worlds.
    - Multiple physics engines (ODE, Bullet, DART). General-purpose robot sim, less specialized on contact accuracy than MuJoCo.
- **Choosing**:
    - Contact-rich control/RL, need speed + stability + derivatives → MuJoCo.
    - ROS-integrated full-robot sim with realistic sensors → Gazebo.
    - GPU-scale parallel RL or photorealistic perception → [[Isaac Lab|Isaac]].

---

## MuJoCo API (from UR10e project)

### Load model and data
```python
import mujoco
import numpy as np

model = mujoco.MjModel.from_xml_path("mujoco_menagerie/universal_robots_ur10e/scene.xml")
data  = mujoco.MjData(model)

# model.nq = 6      (6 joint angles)
# model.nu = 6      (6 actuators / control inputs)
# model.opt.timestep = 0.002   (2ms physics step)
```

### Core simulation loop
```python
# Reset to initial state
mujoco.mj_resetData(model, data)

# Forward kinematics — compute positions from joint angles (no physics step)
mujoco.mj_forward(model, data)

# Apply control and step physics
data.ctrl[:] = torques          # set actuator commands
mujoco.mj_step(model, data)     # advance simulation by one timestep

# Inverse dynamics — compute torques needed for a given acceleration
data_copy.qpos[:] = data.qpos[:]
data_copy.qvel[:] = data.qvel[:]
data_copy.qacc[:] = np.zeros(model.nv)   # zero acceleration → gives gravity+Coriolis
mujoco.mj_inverse(model, data_copy)
passive_forces = data_copy.qfrc_inverse[:6]
```

### Reading EE position via named site
```python
# Sites are defined in MJCF at fixed offsets from bodies — cleaner than body chain
site_id = mujoco.mj_name2id(model, mujoco.mjtObj.mjOBJ_SITE, "attachment_site")
ee_pos  = data.site_xpos[site_id]   # (3,) world-frame position

# Read joint state
joint_pos = data.qpos[:6]   # (6,) joint angles in radians
joint_vel = data.qvel[:6]   # (6,) joint velocities
```

### Rendering
```python
renderer = mujoco.Renderer(model, height=480, width=640)
renderer.update_scene(data, camera="front_camera")
frame = renderer.render()   # numpy uint8 (H, W, 3)

# For animation — collect frames, write to GIF
frames.append(frame)
imageio.mimsave("output.gif", frames, fps=30)
```

---

## PDController in MuJoCo (UR10e project)

```python
class PDController:
    def __init__(self, kp, kd, num_joints=6):
        self.kp = np.full(num_joints, kp)
        self.kd = np.full(num_joints, kd)
        self.target_angles = np.zeros(num_joints)

    def compute_control(self, current_pos, current_vel):
        error = self.target_angles - current_pos
        error_derivative = -current_vel           # target velocity = 0
        return self.kp * error + self.kd * error_derivative

# Used: kp=150, kd=20 for UR10e
controller = PDController(kp=150.0, kd=20.0, num_joints=6)
```

**Torque law:** `τ = Kp·(θ_d − θ) + Kd·(0 − θ̇)`

Kp=150 is the stiffness; Kd=20 damps oscillation. Too high Kp with low Kd → oscillation. Too high Kd → overdamped and slow.

---

## Gravity compensation via `mj_inverse`

For velocity control, feedforward gravity cancellation is mandatory — without it the arm sags:

```python
class VelocityController:
    def compute_control(self, model, data):
        # Get gravity + Coriolis via inverse dynamics at zero acceleration
        data_copy.qpos[:] = data.qpos[:]
        data_copy.qvel[:] = data.qvel[:]
        data_copy.qacc[:] = 0
        mujoco.mj_inverse(model, data_copy)
        passive_forces = data_copy.qfrc_inverse[:6]

        if moving:
            vel_error = target_velocities - data.qvel[:6]
            torque = passive_forces + kd * vel_error
        else:
            # Hold position: PD + gravity comp
            pos_error = initial_positions - data.qpos[:6]
            torque = passive_forces + kp * pos_error + kd * (-data.qvel[:6])
        return torque
```

**Why `mj_inverse`?** Gravity and Coriolis change with configuration. Setting `qacc=0` in inverse dynamics returns exactly the torques needed to hold position — that's the feedforward that cancels gravity. Without it, your PD gains fight gravity instead of executing the motion.

---

## UR10e menagerie (NYU project)

From `mujoco_menagerie/universal_robots_ur10e/scene.xml`:
- 6-DOF serial arm, joints: shoulder_pan, shoulder_lift, elbow, wrist_1, wrist_2, wrist_3
- EE site: `"attachment_site"` — use `data.site_xpos` (not body chain)
- Physics timestep: 2ms
- Implemented four kinematics tasks: FK position, IK position, FK velocity (J·q̇), IK velocity (J⁺·ẋ)

Full note: [[Foundations of Robotics — UR10e Project]]

---

## MuJoCo vs Gazebo decision table

| Criterion | MuJoCo | Gazebo |
|-----------|--------|--------|
| Contact accuracy | ✅ Soft-constraint, stable | Depends on physics plugin |
| Speed | ✅ Very fast | Moderate |
| Derivatives | ✅ Analytical | ❌ |
| ROS2 integration | Manual bridge | ✅ Native plugins |
| Sensor models | Basic | ✅ Rich (lidar, camera, IMU) |
| Asset formats | MJCF / URDF | URDF / SDF |
| RL research | ✅ Default choice | Rarely |

---

## Where I've used it / context

- **UR10e NYU project**: MuJoCo Menagerie, PDController, velocity controller with `mj_inverse` gravity comp, 4 kinematics tasks, rendered as animated GIFs (see [[Foundations of Robotics — UR10e Project]])
- **FENCE-BOT** used [[Isaac Lab|Isaac]] for GPU-parallel sim; MuJoCo is what I'd reach for if contact accuracy and differentiability mattered more than rendering
- On the roadmap as simulation target for the [[MPC & Virtual Fixtures|Kim MPC project]]

---

## Interview follow-ups

- **Q:** Why is MuJoCo popular for control/RL?
    - **A:** Fast, stable, accurate contact via a soft-constraint model that avoids the stiffness/blowups of naive rigid contact, plus analytical derivatives that suit gradient-based control and trajopt.
- **Q:** When Gazebo over MuJoCo?
    - **A:** When I want tight ROS2 integration and realistic sensor models — Gazebo's plugins and URDF/SDF ecosystem fit that. MuJoCo is more of a physics/control research tool.
- **Q:** How do you choose a simulator?
    - **A:** Match strength to problem: MuJoCo for contact-rich control/RL, Gazebo for ROS-integrated sensor-rich sim, Isaac for GPU-parallel RL or photorealistic perception.
- **Q:** How do you read EE position in MuJoCo?
    - **A:** Via a named site: `mujoco.mj_name2id(model, mjOBJ_SITE, "attachment_site")` then `data.site_xpos[site_id]`. Sites are defined in MJCF at a fixed offset from a body — cleaner than computing through the body chain.

## Gotchas / what trips me up

- Calling MuJoCo's contact "rigid" — it's a smooth soft-constraint approximation (that's why it's stable and fast).
- Assuming one simulator is universally best — they're specialized.
- Forgetting gravity compensation in velocity control — arm sags without `mj_inverse` feedforward.
- `data.site_xpos` vs `data.body_xpos` — prefer sites for named EE links (cleaner offset handling).
- `mj_forward` computes kinematics but does not step physics; `mj_step` does both.

## Links

- Related: [[Isaac Lab]], [[Contact Modeling]], [[MPC & Virtual Fixtures]], [[Robot Dynamics Formulations]], [[Foundations of Robotics — UR10e Project]], [[Simulation Environments Comparison]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is MuJoCo known for? ? Fast, stable, accurate contact dynamics via a smooth soft-constraint model (avoids rigid-contact blowups), plus analytical derivatives — the default for control/RL research.

When do you pick Gazebo over MuJoCo? ? For tight ROS2 integration and realistic sensor models in a full-robot system test (URDF/SDF, sensor plugins). MuJoCo is more a physics/control research tool.

How do you choose among MuJoCo / Gazebo / Isaac? ? MuJoCo: contact-rich control/RL. Gazebo: ROS-integrated sensor-rich sim. Isaac: GPU-parallel RL or photorealistic perception.

What is the difference between mj_forward and mj_step in MuJoCo? ? mj_forward computes forward kinematics and dynamics from current state but does not advance time — use it to read positions/velocities after setting qpos/qvel. mj_step advances the simulation by one timestep using data.ctrl as actuator commands.

How do you get gravity compensation in a MuJoCo velocity controller? ? Use mj_inverse with qacc=0 to compute the torques that produce zero acceleration at the current configuration — that's gravity + Coriolis. Add these as feedforward to your PD terms. Without this the arm sags under its own weight and the controller fights gravity instead of executing the motion.

Why use data.site_xpos over data.body_xpos for EE position in MuJoCo? ? Sites are defined in MJCF at a fixed offset from a body — the attachment_site is placed exactly at the EE tip. body_xpos gives the body origin which is usually not the EE. Sites are cleaner and avoid manually computing the offset transform.
