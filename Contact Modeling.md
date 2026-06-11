---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [dynamics, contact, force-sensing, fence-bot, surgical]

# Contact Modeling


---

## Core idea

Contact is where a robot touches the world — and it's notoriously hard because it's **discontinuous and non-smooth**: forces switch on instantly, motion is constrained suddenly, and the equations become stiff/hybrid. It's central to manipulation, locomotion, and — for me — surgical force sensing.

## Key facts & formulas

- **Two separate problems**:
    1. **Contact detection** — _is_ there contact? (collision check, force threshold, distance). Discrete event.
    2. **Contact dynamics** — _what force_ results and how does it constrain motion? (the hard, continuous part).
- **Contact force models**:
    - **Penalty / spring-damper**: force ∝ penetration depth + velocity ($F = k\delta + b\dot\delta$). Simple, smooth, but soft (allows penetration) and stiff numerically.
    - **Rigid / complementarity (LCP)**: no penetration, force via linear complementarity conditions (force ≥ 0, gap ≥ 0, can't both be positive). Accurate but expensive and non-smooth — what MuJoCo's soft-constraint model approximates smoothly.
- **Friction**: Coulomb friction cone — tangential force bounded by μ·normal force. Linearized to a pyramid for LCP solvers.
- **In dynamics**: contact enters the EOM as $J_c^T\lambda$ (contact Jacobian transpose × contact force) — same structure as [[Floating-Base & Whole-Body|floating-base]] and the [[Jacobian|force duality]] $\tau = J^TF$.
- **Rising-edge detection**: detect the _transition_ into contact (force crosses threshold going up), not just the sustained state — debounces noise and catches the instant of impact.


## Links

- Related: [[Jacobian]], [[Robot Dynamics Formulations]], [[Floating-Base & Whole-Body]], [[Lagrangian Neural Networks]], [[PINN Contact Estimation]], [[Force Feedback & Haptics]], [[MuJoCo & Gazebo]]
- Parent: [[00 Knowledge Map]]

---

