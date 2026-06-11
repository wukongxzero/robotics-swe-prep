---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, lqr, optimal-control, deep]

# LQR

> [!star] Interviewer will push here — full derivation below LQR is the "I implemented optimal control on real hardware" story. Know the cost, the Riccati equation, and what Q/R _do_, cold.

> [!question] Explain it cold
> 
> - Write the LQR cost function and say what Q and R trade off.
> - Where does the gain K come from? (Name the equation.)
> - Walk the derivation from cost to Riccati at a high level.

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

## Where I've used it

- **WALL-E V3 gimbal**: ran **LQR** on the 2-axis gimbal at 200 Hz on the Arduino UNO, with Q/R tuned to balance tracking against servo effort — sitting next to pole-placement ($K=[0.72\dots]$) and the [[LQG]] variant. State = angle + rate per axis. Computed the gain offline (solve the Riccati once), shipped the constant $K$ to the MCU — you don't solve Riccati on an ATmega in the loop.

## Interview follow-ups

- **Q:** Where does K come from in LQR?
    - **A:** $K = R^{-1}B^TP$, where $P$ solves the algebraic Riccati equation $A^TP + PA - PBR^{-1}B^TP + Q = 0$. Q and R are my design weights.
- **Q:** How do you tune Q and R?
    - **A:** Q penalizes state error, R penalizes effort — it's the ratio that matters. Start with Bryson's rule (diagonal entries = 1/max-acceptable-value²), then push Q up for tighter tracking or R up to calm actuation. On the gimbal I traded tracking sharpness against servo strain.
- **Q:** Why is LQR stable by construction but pole placement isn't?
    - **A:** LQR minimizes a positive cost with a stabilizing Riccati solution, which guarantees $A-BK$ is Hurwitz. Pole placement can technically place stable poles too, but it gives no guarantee they're _achievable_ without saturating the actuator — LQR builds the effort cost in.
- **Q:** Do you solve Riccati on the embedded target?
    - **A:** No — solve offline, ship the constant gain. The MCU just does $u = -Kx$, which is cheap and deterministic. Re-solving online is for gain-scheduled or adaptive cases.

## Gotchas / what trips me up

- Saying Q/R absolute values matter — it's the _ratio_.
- Forgetting LQR needs full-state feedback; if you can't measure all states you pair it with an estimator → [[LQG]].
- Mixing up CARE (continuous) and DARE (discrete) forms.

## Links

- Related: [[State-Space & Pole Placement]], [[Kalman Filter]], [[LQG]], [[MPC & Virtual Fixtures]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

LQR cost function and what Q and R do? ? J = ∫(xᵀQx + uᵀRu)dt. Q penalizes state error (tighter regulation), R penalizes control effort (gentler actuation); the Q/R ratio is the knob.

LQR optimal gain formula? ? K = R⁻¹BᵀP, where P solves the continuous algebraic Riccati equation AᵀP + PA − PBR⁻¹BᵀP + Q = 0.

Why is LQR closed-loop stable by construction? ? It minimizes a positive-definite cost via the stabilizing Riccati solution, which makes A−BK Hurwitz — and it builds control effort into the cost, unlike hand pole placement.

Do you solve the Riccati equation on the embedded target? ? No — solve offline once, ship the constant gain K; the MCU just computes u = −Kx (cheap, deterministic). Online solving is for adaptive/gain-scheduled cases.