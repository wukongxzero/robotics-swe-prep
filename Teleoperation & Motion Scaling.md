---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, teleoperation, motion-scaling, articulus]

# Teleoperation & Motion Scaling

> [!star] Hands-on (control layer) + domain (surgeon-facing) — flagship pattern The master-slave teleop pattern is the heart of surgical robotics and connects directly to your FENCE-BOT VR pipeline. Provenance: hands-on for the control/comms; domain-level for the surgeon-ergonomics specifics.

> [!question] Explain it cold
> 
> - What is master-slave teleoperation?
> - What does motion scaling do, and why is it essential in surgery?
> - What is tremor filtering and how does it relate to the control loop?

---

## Core idea

Teleoperation maps a human operator's motion (master device) onto a remote/constrained manipulator (slave). In surgery, the surgeon works at a console and the instruments mirror their hands — but with transformations that make superhuman precision possible: **motion scaling** (shrink hand motion) and **tremor filtering** (remove involuntary shake).

## Key facts & formulas

- **Master-slave loop**: master pose → mapping/transform → slave command; optionally slave force → master ([[Force Feedback & Haptics|haptics]]). Same shape as the [[VR Teleop Pipeline|FENCE-BOT pipeline]] (VR master → IK → sim slave).
- **Motion scaling**: slave moves a _fraction_ of master motion (e.g. 3:1 or 5:1 — surgeon moves 3 mm, instrument moves 1 mm). Turns coarse human motion into fine instrument motion — essential for delicate tissue work. A simple gain in the mapping, big clinical effect.
- **Tremor filtering**: low-pass / notch out the ~8–12 Hz physiological hand tremor before commanding the slave. A filtering stage in the teleop pipeline — relates to [[Complementary Filter|sensor filtering]] and adds phase lag you must budget for ([[Frequency-Domain Analysis|phase margin]], [[Latency Budgets]]).
- **Clutching**: decouple master and slave to re-center the master without moving the instrument (like lifting a mouse) — needed because the master workspace is smaller than the task.
- **Latency is safety-critical**: the master-slave loop must be low-latency and deterministic, or the surgeon feels lag/instability — which is _why_ the [[iceoryx Zero-Copy|zero-copy]] / [[Real-Time Determinism|real-time]] work matters here specifically.

## Where I've used it

- **Articulus** (hands-on, control/comms): the real-time stack underneath the master-slave loop — the deterministic, low-latency path that makes scaled teleop feel direct. I built the systems layer; the surgeon-facing scaling ratios / ergonomics are domain knowledge.
- **FENCE-BOT** (hands-on, the pattern): VR master → DLS IK → simulated slave is exactly this loop minus the clinical transforms — a concrete demonstration I can point to.

## Interview follow-ups

- **Q:** Why is motion scaling important in surgery?
    - **A:** It converts coarse hand motion into fine instrument motion — the surgeon moves several mm, the tool moves one — enabling precision beyond unaided human dexterity for delicate tissue. It's a gain in the master-slave mapping with outsized clinical value.
- **Q:** What's tremor filtering and what's its cost?
    - **A:** Filtering out the involuntary ~8–12 Hz hand tremor before commanding the instrument. The cost is added phase lag/latency, which eats into the teleop loop's stability margin — so it has to be budgeted against responsiveness.
- **Q:** Why is latency such a big deal in surgical teleop?
    - **A:** The surgeon is in a closed feedback loop with the instrument (and haptics); lag makes it feel sluggish or unstable and is a safety issue. That's the direct motivation for deterministic, low-latency comms — zero-copy, real-time dispatch.

## Gotchas / what trips me up

- Motion scaling is a mapping gain, not the robot "being slower" — precision, not speed.
- Filtering (tremor) trades latency for smoothness — a [[Frequency-Domain Analysis|phase-margin]] tradeoff, not free.
- Provenance: I owned the control/comms layer; scaling ratios and console ergonomics are domain-level.

## Links

- Related: [[Multi-Arm Laparoscopy]], [[VR Teleop Pipeline]], [[Force Feedback & Haptics]], [[Latency Budgets]], [[Virtual Fixtures]], [[iceoryx Zero-Copy]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is motion scaling and why does surgery need it? ? The slave instrument moves a fraction of the master's motion (e.g. 3:1) — converting coarse hand motion into fine instrument motion for precision beyond unaided dexterity. A mapping gain with large clinical effect.

What is tremor filtering and what does it cost? ? Low-pass/notch removal of the ~8–12 Hz physiological hand tremor before commanding the instrument. Cost: added phase lag/latency that eats teleop stability margin — budgeted against responsiveness.

Why is low, deterministic latency safety-critical in surgical teleop? ? The surgeon is in a closed feedback loop (motion + haptics) with the instrument; lag makes it feel sluggish/unstable — a safety issue. This motivates zero-copy, real-time-deterministic comms.