---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, lqr, optimal-control, deep]

# LQR

> [!star] Interviewer will push here — full derivation below LQR is the "I implemented optimal control on real hardware" story. Know the cost, the Riccati equation, and what Q/R _do_, cold.


---

## Core idea

LQR (Linear Quadratic Regulator) finds the full-state-feedback gain $u = -Kx$ that **minimizes a quadratic cost** balancing state error against control effort, for a linear system. Instead of guessing pole locations ([[State-Space & Pole Placement]]), you specify _what you care about_ via weight matrices and the optimal $K$ falls out.

## The cost

$$J = \int_0^\infty \left( x^T Q x + u^T R u \right) dt$$

- $Q \succeq 0$ — penalizes state error. Bigger $Q$ → tighter regulation, more aggressive.
- $R \succ 0$ — penalizes control effort. Bigger $R$ → lazier, gentler actuation.
- The ratio $Q/R$ is the single knob: how much do I care about holding the state vs. how much do I want to spend doing it.

## Derivation (continuous, infinite-horizon)

1. **Optimal value function is quadratic.** Guess the cost-to-go $V(x) = x^T P x$ for some symmetric $P \succeq 0$.
2. **Hamilton–Jacobi–Bellman.** The HJB condition for the LQR problem is $$\min_u \left[ x^T Q x + u^T R u + \frac{\partial V}{\partial x}^T(Ax + Bu)\right] = 0.$$ With $V = x^T P x$, $\partial V/\partial x = 2Px$.
3. **Minimize over u.** Differentiate the bracket w.r.t. $u$ and set to zero: $$2Ru + 2B^T P x = 0 ;\Rightarrow; u^* = -R^{-1}B^T P, x.$$ So the gain is $\boxed{K = R^{-1}B^T P}$.
4. **Substitute back** → the terms must vanish for all $x$, giving the **Continuous Algebraic Riccati Equation (CARE)**: $$A^T P + P A - P B R^{-1} B^T P + Q = 0.$$
5. **Solve CARE** for the unique stabilizing $P \succeq 0$, then $K = R^{-1}B^TP$.

- **Discrete version** (what you code): minimize $\sum (x_k^TQx_k + u_k^TRu_k)$, solve the **DARE** $$P = A^TPA - A^TPB(R + B^TPB)^{-1}B^TPA + Q,\quad K = (R+B^TPB)^{-1}B^TPA.$$
- **Requirements**: $(A,B)$ controllable (or stabilizable) for a stabilizing solution to exist; $(A,\sqrt{Q})$ observable for uniqueness.
- LQR closed loop is _guaranteed stable_ (poles of $A-BK$ in LHP) — a big advantage over hand pole-placement.


## Links

- Related: [[State-Space & Pole Placement]], [[Kalman Filter]], [[LQG]], [[MPC & Virtual Fixtures]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

