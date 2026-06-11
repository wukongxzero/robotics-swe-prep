---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [dynamics, contact, force-sensing, fence-bot, surgical]

# Contact Modeling

> [!question] Explain it cold
> 
> - Why is contact hard to model/simulate?
> - What's the difference between detecting contact and modeling contact dynamics?
> - How did you detect a hit in FENCE-BOT?

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


## Interview follow-ups

- **Q:** Why is contact hard to simulate?
    - **A:** It's discontinuous and stiff — forces switch on instantly, motion gets suddenly constrained, and rigid contact is a complementarity problem (no penetration, non-negative force) that's non-smooth and expensive. Penalty models smooth it but allow penetration and create stiff ODEs.
- **Q:** How did you detect contact in FENCE-BOT?
    - **A:** A force-magnitude threshold with rising-edge detection — I trigger on the transition into contact rather than the sustained state, which debounces sensor noise and captures the impact instant. Logged in_contact and force_mag to CSV.
- **Q:** How does contact force enter the dynamics?
    - **A:** As $J_c^T\lambda$ — the contact Jacobian transpose maps the contact force into joint space, same duality as $\tau = J^TF$.
- **Q:** Why did the contact LNN underfit?
    - **A:** Contact events were rare (28 samples) and the dynamics are non-smooth/discontinuous — hard for a smooth network to fit from sparse data. The free-flight LNN converged fine (smooth dynamics, MSE ~1.12); contact is the genuinely hard regime.


## Links

- Related: [[Jacobian]], [[Robot Dynamics Formulations]], [[Floating-Base & Whole-Body]], [[Lagrangian Neural Networks]], [[PINN Contact Estimation]], [[Force Feedback & Haptics]], [[MuJoCo & Gazebo]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

The two separate contact problems? ? Detection (is there contact? — discrete event via collision/threshold/distance) and dynamics (what force results and how it constrains motion — the hard continuous/non-smooth part).

Penalty vs rigid (complementarity) contact models? ? Penalty: F = kδ + bδ̇ from penetration — simple/smooth but allows penetration and is numerically stiff. Rigid/LCP: no penetration, force via complementarity (force≥0, gap≥0) — accurate but expensive and non-smooth.

How did you detect contact in FENCE-BOT? ? Force-magnitude threshold with rising-edge hit detection — trigger on the transition into contact, debouncing noise and catching the impact instant; logged in_contact/force_mag to CSV.

Why is contact hard to model, and how did it show in the LNN? ? It's discontinuous and stiff. The contact LNN underfit on 28 sparse samples of non-smooth dynamics, while the free-flight (smooth) LNN converged to MSE ~1.12.