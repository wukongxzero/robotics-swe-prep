---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [optimization, trajectory, collocation, kim]

# Trajectory Optimization


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


## Links

- Related: [[MPC & Virtual Fixtures]], [[Robot Dynamics Formulations]], [[Floating-Base & Whole-Body]], [[acados OCS2 CasADi]]
- Parent: [[00 Knowledge Map]]

---

