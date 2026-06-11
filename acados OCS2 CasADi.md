---

## type: concept domain: Simulation & tooling status: skeleton last-reviewed: tags: [optimization, mpc, acados, casadi, ocs2, kim]

# acados OCS2 CasADi

> [!warning] Research-direction tooling — learning, not yet used in anger These are the MPC/optimal-control toolchains scoped for the [[MPC & Virtual Fixtures|Kim project]] (Spring 2027). Frame as "the toolchain I'm picking up for the virtual-fixtures MPC work," not shipped experience.

> [!question] Explain it cold
> 
> - What does each of the three tools do, and how do they relate?
> - Why do you need a special toolchain for real-time MPC?
> - Where does autodiff fit in?

---

## Core idea

Building real-time [[MPC & Virtual Fixtures|MPC]] / [[Trajectory Optimization|trajectory optimization]] means repeatedly solving constrained nonlinear optimization fast. These three tools cover the stack: **CasADi** to _model_ the problem and get derivatives, **acados** to _solve_ it fast on embedded targets, **OCS2** as a higher-level optimal-control framework for complex (switched/whole-body) systems.

## Key facts & formulas

- **CasADi**: symbolic framework for automatic differentiation + nonlinear optimization modeling. You write the dynamics/cost symbolically; it generates exact gradients/Jacobians/Hessians and interfaces to NLP solvers (IPOPT). The modeling + autodiff layer. Underpins a lot of robotics MPC.
- **acados**: fast embedded solver for nonlinear MPC and optimal control — implements efficient SQP/interior-point with structure-exploiting QP solvers (HPIPM), C-code generation for real-time on-robot solving. The "make MPC fit in the control period" tool. Often takes CasADi-modeled problems.
- **OCS2**: optimal control framework (ETH) for switched systems — legged locomotion, whole-body MPC, SLQ/DDP-family solvers. Higher-level, for complex floating-base/[[Floating-Base & Whole-Body|whole-body]] problems.
- **Relationship**: CasADi models + differentiates → acados solves fast / OCS2 handles complex system structure. They overlap and interoperate rather than being strictly layered.
- **Why special tooling**: generic NLP solvers are too slow for a control loop; these exploit the problem's _structure_ (sparsity, stagewise/banded Hessians from the time horizon) and generate C code to hit real-time. ([[Real-Time Determinism]])

## Where it's going

- **Kim project (planned)**: MPC-based virtual fixtures, simulation-only, using acados/OCS2/CasADi in [[Isaac Lab]]/[[MuJoCo & Gazebo|MuJoCo]]. This note is the toolchain map for that work — honest framing: learning these for the project, not prior production use.

## Interview follow-ups

- **Q:** Why not just use IPOPT/a generic NLP solver for MPC?
    - **A:** Too slow for a control loop. MPC has exploitable structure — a banded/stagewise problem from the time horizon — and embedded solvers like acados exploit that sparsity and generate C code to solve within the control period. CasADi models and differentiates the problem; acados solves it fast.
- **Q:** What does CasADi specifically give you?
    - **A:** Symbolic modeling plus automatic differentiation — exact gradients/Jacobians/Hessians of your dynamics and cost, and interfaces to solvers. It removes hand-derived derivatives, which is where bugs live.
- **Q:** When OCS2?
    - **A:** Complex systems — switched dynamics, legged/whole-body MPC — where you want a framework with SLQ/DDP-family solvers built for that structure rather than rolling it yourself.

## Gotchas / what trips me up

- Treating them as competitors — they interoperate (CasADi models, acados solves).
- Don't overclaim — this is forward-looking toolchain knowledge for the research project.

## Links

- Related: [[MPC & Virtual Fixtures]], [[Trajectory Optimization]], [[Floating-Base & Whole-Body]], [[Real-Time Determinism]], [[Isaac Lab]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does each of CasADi / acados / OCS2 do? ? CasADi: symbolic modeling + autodiff (exact derivatives, NLP interface). acados: fast embedded NMPC solver with C-code gen for real-time. OCS2: optimal-control framework for switched/whole-body systems.

Why can't you use a generic NLP solver for real-time MPC? ? Too slow for the control loop. MPC has exploitable banded/stagewise structure from the time horizon; embedded solvers (acados) exploit sparsity and generate C code to solve within the control period.

What does CasADi's autodiff remove the need for? ? Hand-derived gradients/Jacobians/Hessians — it generates exact derivatives of dynamics and cost symbolically, eliminating a common source of bugs.