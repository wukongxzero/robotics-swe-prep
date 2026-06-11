---

## type: concept domain: Control theory status: drafted last-reviewed: tags: [control, frequency-domain, bode, nyquist, root-locus, classical]

# Frequency-Domain Analysis


---

## Core idea

Classical control analyzes a system through its **transfer function** $G(s)$ — the Laplace-domain ratio of output to input — and asks how the loop responds to sinusoids of every frequency. Stability and robustness fall out of _how the open-loop gain and phase behave near the frequency where gain = 1_. This is the lens you use before/alongside state-space, and it's where "stability margin" is actually defined.

## Transfer functions & the s-plane

- $G(s) = \frac{Y(s)}{U(s)}$ — Laplace transform of the system; $s = \sigma + j\omega$.
- **Poles** (denominator roots) set stability and speed; **zeros** (numerator roots) shape response. Poles in the **left-half plane** (Re < 0) = stable; right-half = unstable; on the $j\omega$ axis = marginal.
- Same poles as the eigenvalues of $A$ in [[State-Space & Pole Placement]] — two views of one system.

## Bode plot

- Two stacked plots vs frequency (log scale): **magnitude** ($20\log_{10}|G(j\omega)|$ in dB) and **phase** (degrees).
- **Gain crossover frequency** $\omega_{gc}$: where magnitude = 0 dB (gain = 1). **Phase crossover** $\omega_{pc}$: where phase = −180°.
- What you read off it:
    - **Phase margin (PM)** = 180° + phase at $\omega_{gc}$. How much extra phase lag before instability. Want ~30–60°; relates to damping/overshoot.
    - **Gain margin (GM)** = how much gain (dB) you can add at $\omega_{pc}$ before |G| hits 1. Want positive, ~6 dB+.
    - **Bandwidth**: roughly $\omega_{gc}$ — sets response speed. Higher bandwidth = faster but noisier, less robust.
- Intuition: a feedback loop goes unstable when, at some frequency, the signal comes back with gain ≥ 1 _and_ phase = −180° (inverted) — it reinforces itself. Margins measure how far you are from that.

## Nyquist

- Polar plot of $G(j\omega)$ as $\omega$ sweeps. **Nyquist criterion**: closed-loop stability from encirclements of the −1 point ($Z = N + P$). The rigorous stability test; handles cases Bode's margins miss (non-minimum-phase, multiple −180° crossings).
- Distance from the −1 point is the geometric meaning of "margin."

## Root locus

- Plots how closed-loop **pole locations move** in the s-plane as a gain $K$ varies from 0→∞. Design tool: pick the $K$ that places poles for the damping/speed you want; see when a branch crosses into the right-half plane (goes unstable).

## Compensators (the design payoff)

- **Lead** compensator: adds phase near $\omega_{gc}$ → improves phase margin / transient response (≈ a filtered D term).
- **Lag** compensator: boosts low-frequency gain → kills steady-state error (≈ an I term) at the cost of some phase.
- **Lead-lag**: both. The classical analog of tuning PID, but done by shaping the frequency response.


## Links

- Related: [[PID Control]], [[State-Space & Pole Placement]], [[LQR]], [[LQG]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

