---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, state-space, pole-placement]

# State-Space & Pole Placement

> [!question] Explain it cold
> 
> - Write the state-space form and say what each matrix is.
> - What do controllability and observability mean, and how do you check them?
> - What does pole placement actually let you choose, and what's its limitation vs LQR?

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


## Interview follow-ups

- **Q:** You can't place all the poles — why?
    - **A:** The system isn't fully controllable — the controllability matrix is rank-deficient, so some modes can't be moved by the input.
- **Q:** Pole placement vs LQR — when do you pick which?
    - **A:** Pole placement when I have specific dynamic targets and a low-order system. LQR when I want a principled tradeoff between state error and control effort rather than hand-guessing pole locations that might demand unrealistic actuation.
- **Q:** Why do you need observability for the gimbal?
    - **A:** I don't measure the full state directly (rate vs angle); I estimate it via the complementary filter / Kalman. That estimation requires the system to be observable from the available outputs.


## Links

- Related: [[LQR]], [[Kalman Filter]], [[PID Control]], [[Complementary Filter]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

State-space form and what each matrix means? ? ẋ = Ax + Bu, y = Cx + Du. A = dynamics, B = input, C = output, D = feedthrough. Closed loop with u = −Kx gives ẋ = (A−BK)x.

How do you check controllability, and what does it gate? ? Rank of [B AB A²B … Aⁿ⁻¹B] must equal n. If rank-deficient, you can't place all closed-loop poles / steer all modes.

What sets the closed-loop dynamics in full-state feedback? ? The eigenvalues (poles) of (A−BK): negative real part = stable, further left = faster, imaginary part = oscillation.

Pole placement's key limitation vs LQR? ? It ignores control cost — you guess pole locations that may demand unrealistic actuation. LQR optimizes a state-vs-effort cost instead.