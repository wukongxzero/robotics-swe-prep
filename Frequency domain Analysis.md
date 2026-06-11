---

## type: concept domain: Control theory status: drafted last-reviewed: tags: [control, frequency-domain, bode, nyquist, root-locus, classical]

# Frequency-Domain Analysis

> [!question] Explain it cold
> 
> - What does a Bode plot show, and what are you looking for on it?
> - Define gain margin and phase margin — where do you read them?
> - Nyquist vs Bode vs root locus — what does each tell you?

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

## Where I've used it / where it connects

- This is the analysis lens for the **stability margins** of any loop, including the gimbal controllers in WALL-E and the real-time control loops at Articulus. Even when I _design_ in state-space (LQR/LQG), I reason about robustness in the frequency domain — which is exactly why the [[LQG]] caveat matters: **LQR guarantees ≥60° phase / ≥6 dB gain margin; LQG can destroy those**, and the only way to see that is a Bode/Nyquist check of the actual loop.
- Pairs with [[PID Control]] (lead/lag ≈ PID terms) and [[State-Space & Pole Placement]] (poles = eigenvalues).

## Interview follow-ups

- **Q:** What's phase margin and why do you care?
    - **A:** 180° plus the open-loop phase at the gain-crossover frequency — the extra phase lag the loop tolerates before instability. It maps to damping/overshoot; ~30–60° is the usual target. Too little → ringing; negative → unstable.
- **Q:** A loop is stable in sim but oscillates on hardware — frequency-domain explanation?
    - **A:** Real-world phase lag (sensor filtering, computation delay, actuator dynamics) ate the phase margin. Delay adds phase lag that grows with frequency, pushing $\omega_{gc}$ past −180°. I'd Bode the loop including the measured delay and add a lead compensator or reduce bandwidth.
- **Q:** Why use Nyquist over Bode?
    - **A:** Nyquist is the rigorous criterion (encirclements of −1) and correctly handles non-minimum-phase systems and loops that cross −180° multiple times, where Bode's simple GM/PM reading can mislead.
- **Q:** What does root locus give you that Bode doesn't?
    - **A:** A direct picture of _where the closed-loop poles go_ as gain changes — so I can pick a gain for target damping and see exactly when a pole crosses into the RHP.

## Gotchas / what trips me up

- GM/PM from a Bode plot can mislead on non-minimum-phase or multi-crossing systems — fall back to Nyquist.
- Forgetting that **pure time delay** adds frequency-dependent phase lag (−ωT) but no gain change — it silently eats phase margin. Huge in real digital control loops.
- Confusing bandwidth (speed) with stability margin (robustness) — pushing bandwidth up usually trades margin away.

## Links

- Related: [[PID Control]], [[State-Space & Pole Placement]], [[LQR]], [[LQG]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does a Bode plot show and what's the key frequency on it? ? Magnitude (dB) and phase (deg) vs frequency on a log scale. The gain-crossover frequency (where magnitude = 0 dB / gain = 1) is where you read phase margin.

Define phase margin and gain margin. ? Phase margin = 180° + open-loop phase at the gain-crossover frequency (extra phase lag before instability). Gain margin = how much gain you can add at the −180° phase-crossover before |G| reaches 1.

Why does a loop stable in sim oscillate on real hardware (frequency-domain view)? ? Real phase lag — sensor filtering, computation delay, actuator dynamics — eats the phase margin. Pure time delay adds −ωT phase lag with no gain change, silently consuming margin.

Nyquist vs Bode — when do you need Nyquist? ? Nyquist (encirclements of −1) is the rigorous criterion and correctly handles non-minimum-phase systems and multiple −180° crossings, where Bode's simple GM/PM can mislead.

What does root locus plot? ? How the closed-loop pole locations move through the s-plane as a gain K varies 0→∞ — used to pick K for target damping and to see when a pole crosses into the right-half plane.

How do lead and lag compensators map to PID? ? Lead adds phase near crossover to improve phase margin/transients (≈ D term); lag boosts low-frequency gain to kill steady-state error (≈ I term).
![[Pasted image 20260525103718.png]]