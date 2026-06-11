---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, mpc, virtual-fixtures, surgical, research, differentiator]

# MPC & Virtual Fixtures

> [!star] Differentiator + research direction This is double-duty: review _and_ prep for the Prof. Kim project (MPC-based virtual fixtures for teleoperated manipulation, simulation-only, Spring 2027). It's also the most surgical-robotics-relevant control note — virtual fixtures are an active-constraint safety concept straight out of the domain.


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


## Links

- Related: [[LQR]], [[Trajectory Optimization]], [[acados OCS2 CasADi]], [[Safety-Critical Architecture]], [[Teleoperation & Motion Scaling]], [[Virtual Fixtures]], [[Isaac Lab]]
- Parent: [[00 Knowledge Map]]

---

