---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, haptics, force-feedback, control]

# Force Feedback & Haptics

> [!note] Mixed provenance — domain + adjacent Force _sensing/estimation_ connects to your PINN contact work (hands-on-adjacent); haptic _rendering_ and the surgeon-facing feel are domain-level. Flag/correct per your actual Articulus exposure.

> [!question] Explain it cold
> 
> - What is haptic feedback and why does it matter in surgery?
> - Why is haptic rendering a control-stability problem?
> - How do you get force information without a tip force sensor?

---

## Core idea

Haptics closes the loop back to the operator's hand: the surgeon _feels_ the forces the instrument exerts on tissue. It restores the touch sense that minimally-invasive surgery removes. The hard parts are (1) _getting_ the force signal and (2) _rendering_ it stably without the loop going unstable.

## Key facts & formulas

- **Why it matters**: without haptics the surgeon relies on vision alone; force feedback prevents over-pressing delicate tissue and improves dexterity. (Many clinical systems actually ship _without_ full haptics because of the stability difficulty — a real, debated tradeoff.)
- **Haptic rendering = a stability problem**: the haptic loop is a closed control loop between operator and environment. High stiffness + latency + discretization can make it **go unstable** (buzz/oscillate). Governed by **passivity** — the rendered system must not generate energy. Tight, low-latency, high-rate (often ~1 kHz) control is required, which is why [[Latency Budgets|latency]] and [[Real-Time Determinism|determinism]] are central.
- **Getting the force**:
    - **Direct sensing**: a force/torque sensor at the tool tip — accurate but hard to sterilize/miniaturize/cost.
    - **Estimation**: infer force from motor currents / joint torques ([[Jacobian|τ = JᵀF]]) or a model-based estimator — no tip sensor. This is exactly the [[PINN Contact Estimation|PINN contact-force]] / [[Contact Modeling|contact-sensing]] direction.
- **Bilateral teleoperation**: force flows both ways (master↔slave); stability of that two-way loop under latency is the classic teleop-haptics research problem (wave variables, passivity controllers).

## Where I've used it / connects

- **FENCE-BOT** (hands-on-adjacent): the [[Contact Modeling|force-threshold contact sensing]] and [[PINN Contact Estimation|PINN]] force estimation are the _sensing_ half of haptics — knowing tool-tissue force, framed explicitly as surgical force sensing.
- **Articulus**: the real-time/latency stack is the substrate a stable haptic loop would need — flag your actual haptics exposure here.

## Interview follow-ups

- **Q:** Why is haptic rendering hard / a control problem?
    - **A:** It's a closed loop between the operator's hand and the environment; stiffness, latency, and discretization can make it actively unstable (oscillate). Stability is governed by passivity — the rendered system mustn't add energy — so it demands tight, low-latency, high-rate control.
- **Q:** How do you get force feedback without a tip force sensor?
    - **A:** Estimate it — from motor currents/joint torques via τ = JᵀF, or a model-based/learned estimator (e.g. a physics-informed network). That's the contact-force-estimation approach, avoiding a hard-to-sterilize tip sensor.
- **Q:** Why do some surgical robots ship without haptics?
    - **A:** Stable, sterile, cost-effective force feedback is genuinely hard — the stability-under-latency problem plus sensor sterilization/cost. Some systems rely on visual force estimation instead; it's an active tradeoff.

## Gotchas / what trips me up

- Treating haptics as "just add a force sensor" — rendering stability is the real difficulty.
- Forgetting it's bidirectional (bilateral) — both directions must be stable under latency.
- Provenance: sensing/estimation ties to my work; rendering/feel is domain-level.

## Links

- Related: [[PINN Contact Estimation]], [[Contact Modeling]], [[Jacobian]], [[Teleoperation & Motion Scaling]], [[Latency Budgets]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Why is haptic rendering a control-stability problem? ? It's a closed loop between operator and environment; stiffness + latency + discretization can make it unstable (oscillate). Stability is governed by passivity (don't add energy), demanding tight low-latency high-rate control.

How do you get force feedback without a tool-tip force sensor? ? Estimate it from motor currents/joint torques (τ = JᵀF) or a model-based/learned estimator (e.g. PINN) — avoiding a hard-to-sterilize, costly tip sensor.

Why do some surgical robots ship without full haptics? ? Stable, sterile, cost-effective force feedback is hard — stability under latency plus sensor sterilization/cost. Some rely on visual force estimation instead; it's an active tradeoff.