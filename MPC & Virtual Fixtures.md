---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, mpc, virtual-fixtures, surgical, research, differentiator]

# MPC & Virtual Fixtures

> [!star] Differentiator + research direction This is double-duty: review _and_ prep for the Prof. Kim project (MPC-based virtual fixtures for teleoperated manipulation, simulation-only, Spring 2027). It's also the most surgical-robotics-relevant control note — virtual fixtures are an active-constraint safety concept straight out of the domain.

> [!question] Explain it cold
> 
> - What is MPC and what does it do that LQR doesn't?
> - What's the receding-horizon idea, and what gets solved each step?
> - What is a virtual fixture, and why is MPC a natural fit for it?

---

## Core idea

**MPC (Model Predictive Control)** optimizes control over a finite future horizon at _every_ timestep: predict the system's trajectory under the model, solve a constrained optimization for the best control sequence, apply only the _first_ control, then re-solve next step with new measurements. The headline feature LQR lacks: **it handles constraints explicitly** (state, input, safety limits).

## Key facts & formulas

- **Receding horizon**: at each step $k$, solve $$\min_{u_{0:N-1}} \sum_{i=0}^{N-1}\left(x_i^TQx_i + u_i^TRu_i\right) + x_N^TQ_Nx_N$$ $$\text{s.t. } x_{i+1}=Ax_i+Bu_i,\quad u_i\in\mathcal{U},\quad x_i\in\mathcal{X}.$$ Apply $u_0$, discard the rest, re-solve next step. The re-solving = feedback.
- **vs LQR**: unconstrained infinite-horizon MPC _is_ LQR. The value MPC adds is **constraints** ($\mathcal{U}, \mathcal{X}$) — actuator limits, no-go regions, safety bounds — that LQR can't encode. Cost is solving a QP/NLP online every step.
- **Linear MPC** → a quadratic program (QP) each step. **Nonlinear MPC** → an NLP, much heavier.
- **Real-time tractability** is the whole engineering problem: warm-starting, condensing, fast QP solvers. This is what the toolchains are for.
- **Toolchain (in scope for the Kim project)**: **acados** (fast embedded NMPC), **OCS2** (optimal control for switched systems / legged + manipulation), **CasADi** (symbolic autodiff + NLP modeling), with **Isaac Lab** / **MuJoCo** for the simulation.

## Virtual fixtures

- A **virtual fixture** is a software-imposed constraint that _guides or restricts_ a teleoperated tool's motion — like a virtual ruler or a no-fly wall the surgeon's instrument can't cross.
    - **Guidance VF**: gently steer the tool along a desired path (anisotropic stiffness).
    - **Forbidden-region VF**: hard constraint keeping the tool out of protected tissue.
- **Why MPC fits**: a virtual fixture _is_ a state constraint $x \in \mathcal{X}_{safe}$. MPC's native ability to enforce constraints over a predictive horizon makes it the principled way to implement fixtures that anticipate violations rather than reacting after the fact — exactly the safety property surgical teleop wants.
- Ties to [[Safety-Critical Architecture]], [[Teleoperation & Motion Scaling]], and [[Force Feedback & Haptics]] — the surgical-robotics cluster.

## Where I've used it / where it's going

- **Articulus context**: teleoperated multi-arm laparoscopy is the domain virtual fixtures live in — active constraints + motion scaling + safety.
- **Prof. Kim project (Spring 2027, planned)**: MPC-based virtual fixtures for teleoperated manipulation, simulation-only, using acados/OCS2/CasADi in Isaac Lab/MuJoCo. This note is the spec-in-miniature.
- **FENCE-BOT** already touched the adjacent pieces — DLS IK, contact sensing, the VR teleop pipeline — which is the practical substrate this builds on.

## Interview follow-ups

- **Q:** MPC vs LQR — when is the extra cost worth it?
    - **A:** When I have hard constraints — actuator saturation, state bounds, safety/no-go regions. LQR can't encode those; MPC enforces them over a predictive horizon. The price is solving a QP/NLP online each step, so it's worth it when constraints matter and I have the compute.
- **Q:** What's "receding horizon"?
    - **A:** Optimize over an N-step future window, apply only the first control, then re-optimize next step with fresh state. The repeated re-solving is what gives feedback and robustness to disturbance.
- **Q:** How would you implement a surgical no-go region?
    - **A:** As a forbidden-region virtual fixture — a state constraint in the MPC keeping the tool tip out of the protected volume across the prediction horizon, so it slows/redirects _before_ contact rather than reacting after.
- **Q:** What makes MPC hard in real time?
    - **A:** Solving the optimization within the control period. Mitigations: linearize to a QP, warm-start from the last solution, condense the problem, use embedded solvers like acados. Nonlinear MPC is the expensive case.

## Gotchas / what trips me up

- Saying MPC is "just better LQR" — it's LQR _plus constraints_ at the cost of online optimization; unconstrained infinite-horizon MPC literally reduces to LQR.
- Underestimating the real-time solve burden — the entire engineering challenge is fitting the QP/NLP in the loop.

## Links

- Related: [[LQR]], [[Trajectory Optimization]], [[acados OCS2 CasADi]], [[Safety-Critical Architecture]], [[Teleoperation & Motion Scaling]], [[Virtual Fixtures]], [[Isaac Lab]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does MPC do that LQR cannot? ? Handle explicit constraints (state, input, safety/no-go regions) by solving a constrained optimization over a finite horizon each step. Unconstrained infinite-horizon MPC reduces to LQR.

What is the receding-horizon principle? ? At each step, optimize control over an N-step future window, apply only the first control, then re-solve next step with new state. The repeated re-solving provides feedback.

What is a virtual fixture, and why is MPC a natural fit? ? A software-imposed motion constraint (guidance path or forbidden region) for a teleoperated tool. MPC fits because a fixture is a state constraint x ∈ X_safe, which MPC enforces predictively — anticipating violations, not reacting.

What's the core real-time challenge of MPC, and typical mitigations? ? Solving the QP/NLP within the control period. Mitigations: linearize to a QP, warm-start, condense, use embedded solvers (acados).