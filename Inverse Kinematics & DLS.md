---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [kinematics, inverse-kinematics, dls, deep, differentiator]

# Inverse Kinematics & DLS

> [!star] You implemented this with λ=0.1 — flagship note DLS IK on the ASEM arm is a concrete "I implemented a numerically-robust algorithm" story. Know the formula, why the damping term exists, and what λ trades off, cold.

> [!question] Explain it cold
> 
> - What does IK solve, and why is it hard (vs FK)?
> - Write the DLS update and explain every term.
> - What does the damping factor λ actually do, and what's the tradeoff?

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


## Interview follow-ups

- **Q:** Why not just use the Jacobian inverse for IK?
    - **A:** It explodes near singularities — J loses rank, the inverse demands enormous joint velocities. DLS adds a λ²I damping term so the matrix stays invertible and joint rates stay bounded, trading a little accuracy for stability.
- **Q:** What does λ do and how do you pick it?
    - **A:** It's the damping factor balancing tracking accuracy against stability. Small λ tracks tightly but jitters near singularities; large λ is smooth but sluggish with steady-state error. I used 0.1 on FENCE-BOT for stable teleop. Adaptive schemes raise λ only as you approach a singularity.
- **Q:** What's the least-squares interpretation?
    - **A:** DLS minimizes $|J\dot q - \dot x|^2 + \lambda^2|\dot q|^2$ — task-error plus a penalty on joint velocity. The penalty is what bounds the solution when J is near-singular.
- **Q:** Multiple IK solutions — how do you choose?
    - **A:** Pick by secondary criteria — closest to current config (minimal motion), joint-limit avoidance, manipulability — often via the null-space of the Jacobian for redundant arms.


## Links

- Related: [[Jacobian]], [[Forward Kinematics]], [[Frames & Rotations]], [[Contact Modeling]], [[VR Teleop Pipeline]], [[Isaac Lab]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Why is IK hard compared to FK? ? It's the nonlinear inverse of FK — can have multiple, one, or no solutions, and rarely a closed form for general arms, so it's usually solved iteratively on the Jacobian.

Write the DLS IK update and name the key term. ? q̇ = Jᵀ(JJᵀ + λ²I)⁻¹ẋ. The λ²I damping term keeps the matrix invertible even at singularities, preventing the joint-velocity blow-up of the pure pseudoinverse.

What does the damping factor λ trade off? ? Accuracy vs stability. Small λ: tight tracking but jittery/unstable near singularities. Large λ: smooth/robust but sluggish with steady-state error. (Used λ=0.1 on FENCE-BOT.)

What least-squares problem does DLS solve? ? min ‖Jq̇ − ẋ‖² + λ²‖q̇‖² — task tracking plus a joint-velocity penalty; the penalty bounds the solution near singularities.