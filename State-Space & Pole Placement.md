---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, state-space, pole-placement]

# State-Space & Pole Placement


---

## Core idea

State-space represents a system as first-order matrix ODEs in terms of an internal state vector $x$:

$$\dot{x} = Ax + Bu, \qquad y = Cx + Du$$

With full-state feedback $u = -Kx$, the closed loop becomes $\dot{x} = (A - BK)x$. The eigenvalues of $(A-BK)$ are the closed-loop **poles** — and they set the dynamics. Pole placement chooses $K$ to put those eigenvalues wherever you want.

## Key facts & formulas

- $A$ = system/dynamics matrix, $B$ = input matrix, $C$ = output matrix, $D$ = feedthrough (usually 0).
- **Poles** (eigenvalues of $A-BK$) govern stability and response: real part < 0 → stable; further left → faster; imaginary part → oscillation.
- **Controllability**: can the input steer the state anywhere? Check rank of $\mathcal{C} = [B\ \ AB\ \ A^2B\ \dots\ A^{n-1}B]$ = $n$ (full). If not fully controllable, you can't place all poles.
- **Observability**: can you reconstruct the state from outputs? Rank of $\mathcal{O} = [C;\ CA;\ CA^2;\ \dots]$ = $n$. Needed to build an observer / [[Kalman Filter]].
- **Pole placement** (e.g. Ackermann): pick desired poles, solve for $K$. Powerful but _arbitrary_ — you're guessing good pole locations by hand.
- **Limitation**: pole placement says nothing about _cost_. Aggressive poles may demand impossible control effort. [[LQR]] fixes this by optimizing a cost instead of guessing poles.


## Links

- Related: [[LQR]], [[Kalman Filter]], [[PID Control]], [[Complementary Filter]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

