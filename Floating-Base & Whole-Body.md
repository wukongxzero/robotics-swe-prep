---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [dynamics, floating-base, whole-body, locomotion, kim]

# Floating-Base & Whole-Body


---

## Core idea

A **fixed-base** manipulator is bolted down — its base frame is the inertial world. A **floating-base** system (legged robot, humanoid, free-flyer) has no such anchor: its base pose is itself part of the state, driven only indirectly through contacts. This adds 6 unactuated DOF and makes the system **underactuated** — you can't command every degree of freedom directly.

## Key facts & formulas

- **Generalized coordinates** split: $q = [q_{base};\ q_{joints}]$, where $q_{base} \in SE(3)$ is the 6-DOF floating base (no motor on it) and $q_{joints}$ are the actuated joints.
- **Floating-base dynamics**: $$M(q)\ddot q + C(q,\dot q)\dot q + g(q) = S^T\tau + J_c^T\lambda$$
    - $S$ — **selection matrix** picking out the actuated joints (the base rows are zero → underactuation made explicit).
    - $J_c^T\lambda$ — **contact forces** ($\lambda$) mapped through the contact Jacobian. The robot moves its base _only_ by pushing on the world through contacts.
- **Underactuation**: #actuators < #DOF. The base can't be driven directly — you control it through contact forces and momentum. This is _the_ defining challenge of locomotion.
- **Whole-body control (WBC)**: coordinate all joints simultaneously to satisfy multiple prioritized tasks (maintain balance / CoM, track a foot or hand trajectory, respect contact and torque limits) — typically as a hierarchical QP each timestep. Connects to [[MPC & Virtual Fixtures|optimization-based control]].
- **Centroidal dynamics / momentum**: a common reduced model — track the robot's center-of-mass and total momentum rather than every link, for tractable balance/gait planning.


## Links

- Related: [[Robot Dynamics Formulations]], [[Trajectory Optimization]], [[Contact Modeling]], [[MPC & Virtual Fixtures]]
- Parent: [[00 Knowledge Map]]

---

