---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, lqg, optimal-control, estimation]

# LQG

> [!question] Explain it cold
> 
> - What is LQG made of?
> - State the separation principle and why it's the key insight.
> - Why do you need LQG instead of plain LQR in the real world?

---

## Core idea

LQG (Linear Quadratic Gaussian) = **[[LQR]] optimal control + [[Kalman Filter]] optimal estimation**, for a linear system with Gaussian noise. LQR assumes you can measure the full state; in reality you measure noisy partial outputs. LQG runs a Kalman filter to estimate the state, then feeds that estimate to the LQR gain: $u = -K\hat{x}$.

## Key facts & formulas

- **Structure**: Kalman filter produces $\hat{x}$ from noisy measurements; LQR gain $K$ acts on $\hat{x}$ rather than the true $x$.
- **Separation principle** (the key result): you can design the LQR gain $K$ and the Kalman gain $L$ **independently**, and the combination is still optimal. The controller design and the estimator design don't interfere. This is why LQG is tractable — two solved problems, not one coupled mess.
- Closed-loop poles of the LQG system = LQR regulator poles ∪ Kalman estimator poles (the two designs' eigenvalues stack).
- **Caveat**: LQG has no guaranteed robustness margins (the famous "LQG has no guaranteed gain/phase margin" result). LQR alone has great margins; adding the estimator can erode them. LQG/LTR (loop transfer recovery) recovers some.

## Where I've used it

- **WALL-E V3 gimbal**: the **LQG** variant = discrete [[Kalman Filter]] for attitude estimation feeding the [[LQR]] gain, at 200 Hz on the Arduino UNO. This is the "principled full stack" version sitting alongside the simpler pole-placement ($K=[0.72\dots]$) and [[Complementary Filter]] implementations. Being able to say "I implemented LQR, LQG, _and_ pole placement on the same plant and compared them" is a strong interview line.

## Interview follow-ups

- **Q:** What's the separation principle and why does it matter?
    - **A:** For a linear-Gaussian system you can design the optimal controller (LQR gain K) and the optimal estimator (Kalman gain L) separately, and the combination remains optimal. It decouples two hard problems into two solved ones — that's what makes LQG practical.
- **Q:** Why not just use LQR?
    - **A:** LQR needs the full state. Real sensors give noisy, partial measurements, so I estimate the state with a Kalman filter and feed the estimate to the LQR gain — that's LQG.
- **Q:** Any downside to LQG vs LQR?
    - **A:** LQR has guaranteed gain/phase margins; LQG does not — the estimator can erode robustness. LQG/LTR can recover margins by tuning the filter.

## Gotchas / what trips me up

- Stating LQG inherits LQR's robustness — it doesn't; that's the classic gotcha.
- Forgetting LQG = LQR + Kalman; if asked "what's in it," name both halves and the separation principle.

## Links

- Related: [[LQR]], [[Kalman Filter]], [[State-Space & Pole Placement]], [[Complementary Filter]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is LQG composed of? ? LQR optimal full-state-feedback control + Kalman optimal state estimation. The Kalman estimate x̂ feeds the LQR gain: u = −Kx̂.

State the separation principle. ? For a linear-Gaussian system the LQR controller gain and the Kalman estimator gain can be designed independently, and the combination is still optimal — decoupling control and estimation.

What robustness gotcha does LQG have vs LQR? ? LQR has guaranteed gain/phase margins; LQG has none — adding the estimator can erode robustness. LQG/LTR recovers some.