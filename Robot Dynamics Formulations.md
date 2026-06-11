---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [dynamics, lagrangian, newton-euler, hamiltonian, kim]

# Robot Dynamics Formulations

> [!question] Explain it cold
> 
> - Write the manipulator equation of motion and name each term.
> - Lagrangian vs Newton–Euler vs Hamiltonian — what's each good for?
> - What's the difference between forward and inverse dynamics?

---

## Core idea

Dynamics relates **forces/torques ↔ motion** (kinematics ignored forces; dynamics adds mass, inertia, gravity). Three equivalent formulations derive the same equations of motion from different starting points: energy (Lagrangian), force balance (Newton–Euler), or phase-space (Hamiltonian). Prof. Kim's Advanced Robotics covered all three plus floating-base.

## The manipulator equation (the one to know)

$$M(q)\ddot{q} + C(q,\dot{q})\dot{q} + g(q) = \tau$$

- $M(q)$ — **mass/inertia matrix** (configuration-dependent, symmetric positive-definite).
- $C(q,\dot{q})\dot{q}$ — **Coriolis and centrifugal** terms (velocity-dependent coupling).
- $g(q)$ — **gravity** torque.
- $\tau$ — applied joint torques (add $J^TF_{ext}$ for external/contact forces — links to [[Jacobian]] and [[Contact Modeling]]).

## The three formulations

- **Lagrangian** ($\frac{d}{dt}\frac{\partial L}{\partial \dot q} - \frac{\partial L}{\partial q} = \tau$, with $L = T - V$): energy-based (kinetic minus potential). Systematic, great for _deriving_ the symbolic equations of motion, scalar bookkeeping. What I used (via SymPy) for the LNN work. Best for analysis and getting clean $M, C, g$.
- **Newton–Euler**: force/torque balance on each link, done **recursively** (outward for velocities/accelerations, inward for forces). Computationally efficient — the standard for _real-time inverse dynamics_ (O(n)). Best for fast numerical computation.
- **Hamiltonian** ($H = T + V$, work in position+momentum phase space): energy-conserving structure, nice for analysis, optimal control, and energy-based/structure-preserving methods. Underlies some learned-dynamics and symplectic-integrator work.

## Forward vs inverse dynamics

- **Inverse dynamics**: given desired motion $(q,\dot q,\ddot q)$, find required $\tau$. Used for feedforward control / computed-torque. Newton–Euler does this in O(n).
- **Forward dynamics**: given $\tau$, find resulting $\ddot q$ (solve $M^{-1}(\tau - C\dot q - g)$). Used for **simulation** (then integrate). This is what physics engines like [[MuJoCo & Gazebo|MuJoCo]] / Isaac do.

## Where I've used it

- **FENCE-BOT / Jordan's LNN**: derived the Lagrangian dynamics symbolically (SymPy, `solve_ivp`) for the 2D rigid-body + pendulum collision sim — and the [[Lagrangian Neural Networks|LNN]] bugs I fixed (StandardScaler breaking Euler–Lagrange consistency, discrete collision input corrupting Hessians) are _exactly_ failures of preserving this structure in a learned model.
- **Prof. Kim coursework**: Lagrange / Newton–Euler / Hamiltonian, floating-base, whole-body — the theoretical backbone for the [[MPC & Virtual Fixtures|MPC project]].

## Interview follow-ups

- **Q:** Write the equation of motion and explain the terms.
    - **A:** $M(q)\ddot q + C(q,\dot q)\dot q + g(q) = \tau$ — inertia times acceleration, plus velocity-dependent Coriolis/centrifugal, plus gravity, equals applied torque. External forces enter as $J^TF$.
- **Q:** Lagrangian vs Newton–Euler — when each?
    - **A:** Lagrangian (energy-based) to _derive_ clean symbolic equations and for analysis; Newton–Euler (recursive force balance) for _efficient real-time_ computation — it's the O(n) workhorse for inverse dynamics.
- **Q:** Forward vs inverse dynamics?
    - **A:** Inverse: motion → required torque (control/feedforward). Forward: torque → resulting acceleration (simulation). Simulators do forward dynamics then integrate.

## Gotchas / what trips me up

- Saying the three formulations give different physics — they give the _same_ EOM, different derivation paths and computational profiles.
- Mixing up forward (sim) and inverse (control) dynamics directions.
- Forgetting $M$ is configuration-dependent (not constant).

## Links

- Related: [[Jacobian]], [[Lagrangian Neural Networks]], [[Floating-Base & Whole-Body]], [[Contact Modeling]], [[MPC & Virtual Fixtures]], [[Trajectory Optimization]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Write the manipulator equation of motion and name the terms. ? M(q)q̈ + C(q,q̇)q̇ + g(q) = τ: inertia matrix × acceleration + Coriolis/centrifugal + gravity = applied torque (external forces add JᵀF).

Lagrangian vs Newton–Euler — when do you use each? ? Lagrangian (energy, T−V) to derive clean symbolic EOM and for analysis; Newton–Euler (recursive force balance) for efficient O(n) real-time inverse dynamics.

Forward vs inverse dynamics? ? Inverse: desired motion → required torque (control/feedforward). Forward: torque → acceleration q̈ = M⁻¹(τ − Cq̇ − g) (simulation, then integrate).

Why do all three formulations agree? ? They derive the same equations of motion from different starting points (energy / force balance / phase space); they differ in derivation path and computational efficiency, not physics.