---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, haptics, force-feedback, control]

# Force Feedback & Haptics

> [!note] Mixed provenance — domain + adjacent Force _sensing/estimation_ connects to your PINN contact work (hands-on-adjacent); haptic _rendering_ and the surgeon-facing feel are domain-level. Flag/correct per your actual Articulus exposure.


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


## Links

- Related: [[PINN Contact Estimation]], [[Contact Modeling]], [[Jacobian]], [[Teleoperation & Motion Scaling]], [[Latency Budgets]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

