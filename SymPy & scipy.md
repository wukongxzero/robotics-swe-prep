---

## type: concept domain: Simulation & tooling status: drafted last-reviewed: tags: [tooling, sympy, scipy, simulation, fence-bot]

# SymPy & scipy


---

## Core idea

The Python scientific stack split: **SymPy** does _symbolic_ math (derive equations exactly, in closed form), **scipy** does _numeric_ math (integrate, optimize, solve numerically). The classic workflow: derive the dynamics symbolically with SymPy, then integrate them numerically with scipy.

## Key facts & formulas

- **SymPy**: symbolic algebra/calculus — differentiate, simplify, solve equations in closed form. For robotics: derive the [[Robot Dynamics Formulations|Lagrangian and equations of motion]] symbolically (compute $\partial L/\partial q$, etc.), then `lambdify` to fast numeric functions.
- **scipy**:
    - `integrate.solve_ivp` — numerically integrate an ODE/IVP given $\dot x = f(t,x)$ and initial conditions; adaptive steppers (RK45, etc.). This is how you _simulate_ a derived dynamics model forward in time.
    - `optimize`, `linalg`, `signal` — numeric optimization, linear algebra, signal processing.
- **The workflow**: SymPy derives $M(q)\ddot q + C\dot q + g = \tau$ symbolically → lambdify to a numeric RHS → `solve_ivp` integrates it → trajectory data. Exact derivation, fast simulation.


## Links

- Related: [[Lagrangian Neural Networks]], [[Robot Dynamics Formulations]], [[Contact Modeling]]
- Parent: [[00 Knowledge Map]]

---

