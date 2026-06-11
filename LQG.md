---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, lqg, optimal-control, estimation]

# LQG


---

## Core idea

LQG (Linear Quadratic Gaussian) = **[[LQR]] optimal control + [[Kalman Filter]] optimal estimation**, for a linear system with Gaussian noise. LQR assumes you can measure the full state; in reality you measure noisy partial outputs. LQG runs a Kalman filter to estimate the state, then feeds that estimate to the LQR gain: $u = -K\hat{x}$.

## Key facts & formulas

- **Structure**: Kalman filter produces $\hat{x}$ from noisy measurements; LQR gain $K$ acts on $\hat{x}$ rather than the true $x$.
- **Separation principle** (the key result): you can design the LQR gain $K$ and the Kalman gain $L$ **independently**, and the combination is still optimal. The controller design and the estimator design don't interfere. This is why LQG is tractable — two solved problems, not one coupled mess.
- Closed-loop poles of the LQG system = LQR regulator poles ∪ Kalman estimator poles (the two designs' eigenvalues stack).
- **Caveat**: LQG has no guaranteed robustness margins (the famous "LQG has no guaranteed gain/phase margin" result). LQR alone has great margins; adding the estimator can erode them. LQG/LTR (loop transfer recovery) recovers some.


## Links

- Related: [[LQR]], [[Kalman Filter]], [[State-Space & Pole Placement]], [[Complementary Filter]]
- Parent: [[00 Knowledge Map]]

---

