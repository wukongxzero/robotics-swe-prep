---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [kinematics, inverse-kinematics, dls, deep, differentiator]

# Inverse Kinematics & DLS

> [!star] You implemented this with λ=0.1 — flagship note DLS IK on the ASEM arm is a concrete "I implemented a numerically-robust algorithm" story. Know the formula, why the damping term exists, and what λ trades off, cold.


---

## Core idea

Inverse kinematics maps **desired end-effector pose → joint values**. The inverse of [[Forward Kinematics|FK]]. It's hard because it's nonlinear, can have multiple solutions, one, or none, and rarely has a clean closed form for general arms — so you usually solve it **numerically** by iterating on the [[Jacobian]]. **Damped Least Squares (DLS)** is the standard numerically-robust method.

## Analytical vs numerical

- **Analytical (closed-form)**: solve the geometry directly. Fast and exact, but only exists for specific arm geometries (e.g. a spherical wrist). Brittle to design changes.
- **Numerical (iterative)**: start from current $q$, compute task error, step joints to reduce it, repeat. General-purpose, handles any arm, but iterative and can struggle near singularities — which is exactly what DLS fixes.

## The methods, building up

- **Jacobian transpose**: $\dot{q} = \alpha J^T \dot{x}$. Cheap, no inverse, gradient-descent flavor — stable but slow/imprecise.
- **Jacobian (pseudo)inverse**: $\dot{q} = J^{+}\dot{x}$. Fast convergence, but **blows up near singularities** ($J$ near rank-deficient → huge joint velocities).
- **Damped Least Squares (Levenberg–Marquardt)** — the fix: $$\dot{q} = J^T\left(JJ^T + \lambda^2 I\right)^{-1}\dot{x}$$
    - Adds $\lambda^2 I$ inside the inverse so the matrix is **always invertible**, even at singularities — no explosion.
    - **λ (damping factor)** trades accuracy vs stability: small λ → accurate but jittery/unstable near singularities; large λ → smooth and robust but sluggish and with steady-state error. **I used λ = 0.1** on FENCE-BOT — a moderate value favoring stability for the teleop sweep.
    - It's literally solving $\min_{\dot q} |J\dot q - \dot x|^2 + \lambda^2|\dot q|^2$ — least-squares task tracking with a penalty on joint speed (the second term is what keeps it bounded near singularities).


## Links

- Related: [[Jacobian]], [[Forward Kinematics]], [[Frames & Rotations]], [[Contact Modeling]], [[VR Teleop Pipeline]], [[Isaac Lab]]
- Parent: [[00 Knowledge Map]]

---

