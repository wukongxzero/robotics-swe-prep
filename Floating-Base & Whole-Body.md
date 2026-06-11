---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [dynamics, floating-base, whole-body, locomotion, kim]

# Floating-Base & Whole-Body

> [!question] Explain it cold
> 
> - What makes a system "floating-base" and why does it complicate dynamics?
> - What is underactuation and why is it the core challenge?
> - What does whole-body control coordinate?

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

## Where I've used it

- **Prof. Kim's Advanced Robotics**: gait/locomotion, whole-body kinematics, floating-base dynamics, the BSB (balance) stability framework, trajectory optimization. This is the theoretical context for the embodied-AI / optimization direction of the [[MPC & Virtual Fixtures|Kim research project]].
- Not in my hands-on robot work (my arms were fixed-base), so this is **coursework/theory depth** — I'd frame it honestly as such.

## Interview follow-ups

- **Q:** What's different about floating-base vs fixed-base dynamics?
    - **A:** The base pose becomes part of the state and is unactuated — you add 6 DOF you can't directly command, plus contact-force terms ($J_c^T\lambda$). The robot drives its base only through contacts, which is why locomotion is hard.
- **Q:** What is underactuation?
    - **A:** Fewer actuators than degrees of freedom — captured by the selection matrix $S$ zeroing the base rows. You can't independently command every DOF, so you control the unactuated ones indirectly via contact forces and momentum.
- **Q:** What does whole-body control do?
    - **A:** Solves for joint commands satisfying multiple prioritized objectives at once — balance/CoM, end-effector tasks, contact and limit constraints — usually a hierarchical QP per control step.

## Gotchas / what trips me up

- Forgetting the base is unactuated — the selection matrix $S$ is where that lives.
- Conflating whole-body _control_ (instantaneous QP) with trajectory _optimization_ (whole motion over time).

## Links

- Related: [[Robot Dynamics Formulations]], [[Trajectory Optimization]], [[Contact Modeling]], [[MPC & Virtual Fixtures]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What makes a system floating-base and why is it harder? ? No fixed anchor — the base pose (6 DOF in SE(3)) is part of the state and unactuated. The robot moves its base only through contact forces, adding J_cᵀλ terms and underactuation.

What is underactuation, and where does it show up in the equations? ? Fewer actuators than DOF. In floating-base dynamics the selection matrix S zeros the base rows (Sᵀτ), making the base uncommandable directly — controlled via contacts/momentum.

What does whole-body control coordinate? ? All joints at once to meet multiple prioritized tasks (balance/CoM, end-effector tracking, contact & limit constraints), typically a hierarchical QP each timestep.