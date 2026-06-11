---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [kinematics, jacobian, singularities, deep]

# Jacobian

> [!star] You implemented this — own it cold The Jacobian is the hinge between FK and IK and the thing the DLS IK actually operates on. Interviewers love it because it connects velocity, singularities, forces, and IK in one object.


---

## Core idea

The manipulator Jacobian $J(q)$ is the matrix that maps **joint velocities → end-effector velocity**: $$\dot{x} = J(q),\dot{q}$$ where $\dot{x} = [v;\ \omega]$ is the 6-vector of end-effector linear+angular velocity and $\dot{q}$ is the joint-rate vector. It's the first derivative of [[Forward Kinematics|FK]] — the local linear approximation of how the tool moves when you move the joints.

## Key facts & formulas

- **Shape**: $6 \times n$ for an $n$-joint arm (6 = 3 linear + 3 angular task-space DOF). Each **column** = the end-effector velocity contribution of one joint. **Top 3 rows** = linear velocity, **bottom 3** = angular.
- **Per-joint columns** (geometric Jacobian):
    - Revolute joint $i$: linear part $z_{i-1} \times (p_n - p_{i-1})$, angular part $z_{i-1}$.
    - Prismatic joint $i$: linear part $z_{i-1}$, angular part $0$.
- **Singularity**: a configuration where $J$ loses rank — the arm can't move the tool in some direction regardless of joint speed (or needs infinite joint speed). Detect via $\det(JJ^T)=0$ or small singular values. Examples: full extension (boundary), wrist alignment (internal). This is _why_ plain inverse-Jacobian IK blows up and you need [[Inverse Kinematics & DLS|damping]].
- **Manipulability** (Yoshikawa): $w = \sqrt{\det(JJ^T)}$ — a scalar "how far from singular / how freely can I move" measure. The manipulability ellipsoid visualizes directions of easy vs hard motion.
- **Force–torque duality (statics)**: $\tau = J^T F$. Joint torques needed to exert end-effector wrench $F$ are the Jacobian-transpose times the wrench. The _same_ matrix relates velocities (forward) and forces (transpose) — elegant and heavily asked.


## Links

- Related: [[Forward Kinematics]], [[Inverse Kinematics & DLS]], [[Frames & Rotations]], [[Contact Modeling]], [[Robot Dynamics Formulations]]
- Parent: [[00 Knowledge Map]]

---

