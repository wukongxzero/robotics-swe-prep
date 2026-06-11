---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [optimization, trajectory, collocation, kim]

# Trajectory Optimization

> [!question] Explain it cold
> 
> - What does trajectory optimization solve, and how does it differ from MPC?
> - Shooting vs collocation — what's the tradeoff?
> - Why is it the backbone of modern motion planning for dynamic systems?

---

## Core idea

Trajectory optimization finds an **entire optimal motion** — a sequence of states and controls over time — that minimizes a cost (energy, time, smoothness) subject to the dynamics and constraints. Where [[MPC & Virtual Fixtures|MPC]] re-solves a short horizon every step, trajectory optimization typically solves the _whole_ motion offline (and MPC is essentially trajectory optimization run in a receding-horizon loop).

## Key facts & formulas

- **General form**: $$\min_{x(\cdot),u(\cdot)} \int_0^T \ell(x,u),dt + \phi(x_T) \quad \text{s.t. } \dot x = f(x,u),\ \ x\in\mathcal X,\ u\in\mathcal U$$ dynamics as constraints, plus boundary conditions and path constraints.
- **Direct methods** (discretize then optimize — "first discretize, then optimize") turn it into a finite nonlinear program (NLP):
    - **Direct shooting**: optimize over controls only; integrate (roll out) the dynamics to get states. Few variables, but sensitive — small early control changes blow up late states (ill-conditioned for long horizons).
    - **Direct collocation**: optimize over states _and_ controls at knot points, enforce dynamics as **defect constraints** between knots (e.g. Hermite–Simpson). More variables but far better conditioned and robust — the standard for hard problems. "Collocation" = make the polynomial segments agree with the dynamics at collocation points.
    - **Multiple shooting**: hybrid — short shooting segments stitched with continuity constraints.
- **Indirect methods**: derive optimality conditions (Pontryagin / calculus of variations) first, then solve. Elegant, accurate, but hard to set up and brittle — less common in practice.
- **Solvers/tools**: NLP solvers (IPOPT, SNOPT) via modeling layers — [[acados OCS2 CasADi|CasADi]] for autodiff + NLP, acados for fast embedded, OCS2 for switched-system/whole-body.

## Where I've used it / context

- **Prof. Kim coursework**: trajectory optimization alongside floating-base dynamics and gait — the planning layer above the dynamics. Direct context for the [[MPC & Virtual Fixtures|MPC virtual-fixtures project]] (MPC = receding-horizon trajopt).
- **Theory/tooling depth** rather than a shipped implementation — frame honestly, but it's the conceptual bridge between my dynamics knowledge and the optimization-based control direction.

## Interview follow-ups

- **Q:** Trajectory optimization vs MPC?
    - **A:** Trajopt solves a full motion over a horizon (often offline) once; MPC solves a shorter horizon repeatedly online, applying the first control and re-solving — MPC is essentially trajopt in a receding-horizon feedback loop.
- **Q:** Shooting vs collocation — which and why?
    - **A:** Shooting optimizes controls and rolls out dynamics — few variables but ill-conditioned over long horizons. Collocation optimizes states and controls together with dynamics enforced as defect constraints between knots — more variables but much better conditioned, so it's the default for hard/long problems.
- **Q:** How do the dynamics enter the optimization?
    - **A:** As equality constraints — either via integration (shooting) or as defect constraints linking consecutive knot points (collocation). The optimizer can't violate physics.

## Gotchas / what trips me up

- Calling collocation "more accurate" — it's better _conditioned/robust_; both are direct methods approximating the same problem.
- Forgetting shooting's long-horizon sensitivity is the reason collocation exists.

## Links

- Related: [[MPC & Virtual Fixtures]], [[Robot Dynamics Formulations]], [[Floating-Base & Whole-Body]], [[acados OCS2 CasADi]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Trajectory optimization vs MPC? ? Trajopt solves a full optimal motion over a horizon (often offline) once; MPC re-solves a shorter horizon online each step, applying the first control. MPC is trajopt in a receding-horizon loop.

Direct shooting vs direct collocation? ? Shooting: optimize controls, integrate to get states — few variables but ill-conditioned over long horizons. Collocation: optimize states+controls with dynamics as defect constraints between knots — more variables, far better conditioned.

How do dynamics enter a direct trajectory optimization? ? As constraints — via integration (shooting) or defect constraints between knot points (collocation) — so the optimizer cannot violate the physics.