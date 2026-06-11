---

## type: concept domain: Simulation & tooling status: drafted last-reviewed: tags: [tooling, sympy, scipy, simulation, fence-bot]

# SymPy & scipy

> [!question] Explain it cold
> 
> - SymPy vs scipy — symbolic vs numeric, when each?
> - How do you derive equations of motion symbolically then simulate them?
> - What does solve_ivp do?

---

## Core idea

The Python scientific stack split: **SymPy** does _symbolic_ math (derive equations exactly, in closed form), **scipy** does _numeric_ math (integrate, optimize, solve numerically). The classic workflow: derive the dynamics symbolically with SymPy, then integrate them numerically with scipy.

## Key facts & formulas

- **SymPy**: symbolic algebra/calculus — differentiate, simplify, solve equations in closed form. For robotics: derive the [[Robot Dynamics Formulations|Lagrangian and equations of motion]] symbolically (compute $\partial L/\partial q$, etc.), then `lambdify` to fast numeric functions.
- **scipy**:
    - `integrate.solve_ivp` — numerically integrate an ODE/IVP given $\dot x = f(t,x)$ and initial conditions; adaptive steppers (RK45, etc.). This is how you _simulate_ a derived dynamics model forward in time.
    - `optimize`, `linalg`, `signal` — numeric optimization, linear algebra, signal processing.
- **The workflow**: SymPy derives $M(q)\ddot q + C\dot q + g = \tau$ symbolically → lambdify to a numeric RHS → `solve_ivp` integrates it → trajectory data. Exact derivation, fast simulation.

## Where I've used it

- **FENCE-BOT / Jordan's collision sim**: the 2D rigid-body + pendulum simulator used **SymPy for the Lagrangian derivation** and **scipy `solve_ivp`** to integrate the dynamics — generating the training data for the [[Lagrangian Neural Networks|LNN]]. So the symbolic ground-truth dynamics (SymPy) produced trajectories (solve_ivp) that the LNN then tried to learn — and the [[Lagrangian Neural Networks|StandardScaler/Hessian bugs]] I fixed were about preserving that symbolic structure in the learned model.

## Interview follow-ups

- **Q:** SymPy vs scipy — when each?
    - **A:** SymPy for exact symbolic derivation (equations of motion, closed-form derivatives); scipy for numeric work (integrating those dynamics, optimization, linear algebra). Derive symbolically, simulate numerically.
- **Q:** How do you go from a Lagrangian to a simulation?
    - **A:** SymPy derives the equations of motion symbolically, lambdify converts them to a fast numeric function, then scipy's solve_ivp integrates that ODE from initial conditions to get a trajectory.
- **Q:** What's solve_ivp?
    - **A:** scipy's initial-value-problem integrator — give it ẋ = f(t,x) and x₀, it adaptively steps (RK45 etc.) to produce the trajectory. The numeric simulation workhorse.

## Gotchas / what trips me up

- Forgetting to lambdify — symbolic expressions are slow to evaluate in a loop.
- solve_ivp stiffness — contact/stiff dynamics need a stiff solver (Radau/BDF), not RK45.

## Links

- Related: [[Lagrangian Neural Networks]], [[Robot Dynamics Formulations]], [[Contact Modeling]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

SymPy vs scipy? ? SymPy: symbolic/exact math (derive equations of motion, closed-form derivatives). scipy: numeric (integrate ODEs, optimize, linalg). Derive symbolically, simulate numerically.

How do you go from a Lagrangian to a forward simulation? ? SymPy derives the equations of motion symbolically → lambdify to a fast numeric RHS → scipy solve_ivp integrates it from initial conditions → trajectory.

What is scipy's solve_ivp? ? An initial-value ODE integrator: given ẋ = f(t,x) and x₀, it adaptively steps (RK45 etc.) to produce the trajectory. Use a stiff solver (Radau/BDF) for stiff/contact dynamics.