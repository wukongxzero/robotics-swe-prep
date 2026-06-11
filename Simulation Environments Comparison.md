---

## type: concept domain: Simulation & Tooling status: drafted last-reviewed: tags: [sim, gazebo, isaac, mujoco, ros2]

# Simulation Environments Comparison

> [!question] Explain it cold
>
> - What simulator do you reach for and why?
> - Why is Gazebo bad for physics-heavy tasks?
> - What's the GPU-parallelism advantage of Isaac Lab over everything else?

---

## The landscape

| Simulator | Physics Engine | GPU Parallel | Rendering | Best For |
|-----------|---------------|-------------|-----------|----------|
| **Isaac Sim / Isaac Lab** | PhysX (NVIDIA) | Yes — thousands of envs | Photorealistic (ray-traced) | RL training, sim-to-real perception, high-fidelity manipulation |
| **MuJoCo** | Own (fast, accurate contacts) | Partial via MJX (JAX) | Basic | Control research, accurate contact dynamics, academic RL |
| **Gazebo Classic** | ODE / Bullet | No | Basic | Legacy ROS integration, nav stack testing |
| **Gazebo Harmonic** | Bullet / DART / custom | No | Improved | ROS2 Jazzy native sim, full-stack ROS2 testing |
| **PyBullet** | Bullet | No | Basic | Quick prototyping (deprecated/unmaintained) |
| **Webots** | ODE | No | Decent | Education, lightweight ROS2 |

---

## Key facts & formulas

**Why Isaac Lab wins for RL:**
- Runs **thousands of parallel envs on a single GPU** — same PhysX simulation vectorized across `num_envs` instances
- Orders of magnitude more samples/sec than CPU sims → RL that would take days on MuJoCo/Gazebo trains in hours
- Built on **Omniverse / USD**: scene described in Universal Scene Description, assets fully programmable

**Why Gazebo is painful for physics-heavy tasks:**
- ODE contact solver is numerically weak — objects drift, slip, behave unrealistically under load
- No GPU acceleration → 1 env at a time, real-time only
- Fine for: nav testing, SLAM, full ROS2 stack integration (sensors, tf, nav2)
- Bad for: anything requiring accurate force/torque, RL data generation

**MuJoCo's advantage:**
- Best-in-class **contact solver** (complementarity-based, numerically stable)
- Fastest per-step wall time for single-env control research
- MJX (MuJoCo on JAX) enables GPU batching but less mature than Isaac
- No photorealistic rendering → sim-to-real gap for perception tasks

**Gazebo Harmonic (new):**
- Native ROS2 Jazzy integration — replaces Gazebo Classic
- Still CPU-only, physics still mediocre
- Right choice when you need the full ROS2 sensor/tf/nav2 stack to work out of the box

---

## Decision tree

```
Need RL training or massive data generation?
  └─ Yes → Isaac Lab (GPU parallel)

Need accurate contact/manipulation research?
  └─ Yes → MuJoCo

Need full ROS2 stack (Nav2, SLAM, tf, sensors)?
  └─ Yes → Gazebo Harmonic

Need photorealistic rendering for perception sim-to-real?
  └─ Yes → Isaac Sim

Quick prototype or learning?
  └─ MuJoCo or Isaac Lab
```

---

## Where I've used each

- **Isaac Lab**: FENCE-BOT — 6-DOF arm simulation, DLS IK validation, contact sensing, VR teleop target over UDP
- **Gazebo**: Early ROS2 testing before switching to Isaac for physics fidelity
- **Manual URDF**: All robots — understanding the full model before feeding any simulator

---


---

## Links

- Related: [[Isaac Lab]], [[MuJoCo & Gazebo]], [[URDF & CAD Pipeline]], [[ROS2 Comm Patterns]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Which simulator do you use for RL training and why? ? Isaac Lab — GPU-parallel simulation runs thousands of envs simultaneously on one GPU (PhysX vectorized), giving orders of magnitude more samples/sec than CPU sims like MuJoCo or Gazebo.

Why is Gazebo bad for contact-heavy tasks? ? Its ODE physics solver is numerically weak — objects drift and behave unrealistically under load. Fine for nav/SLAM/ROS2 stack testing, but invalid results for force/torque or RL training.

MuJoCo vs Isaac Lab — when do you pick each? ? MuJoCo: accurate contact dynamics, control research, fast single-env stepping. Isaac Lab: RL at scale (GPU parallel), photorealistic sim-to-real, high-fidelity manipulation with thousands of envs.

What is Gazebo Harmonic and when do you use it? ? The new ROS2-native Gazebo (replaces Classic), integrated with ROS2 Jazzy. Use it when you need the full ROS2 sensor/tf/nav2 stack. Still CPU-only with mediocre physics.
