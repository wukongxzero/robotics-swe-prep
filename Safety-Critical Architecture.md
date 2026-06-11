---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, teleoperation, motion-scaling, articulus]

# Teleoperation & Motion Scaling

> [!star] Hands-on (control layer) + domain (surgeon-facing) — flagship pattern The master-slave teleop pattern is the heart of surgical robotics and connects directly to your FENCE-BOT VR pipeline. Provenance: hands-on for the control/comms; domain-level for the surgeon-ergonomics specifics.


---

## Core idea

Teleoperation maps a human operator's motion (master device) onto a remote/constrained manipulator (slave). In surgery, the surgeon works at a console and the instruments mirror their hands — but with transformations that make superhuman precision possible: **motion scaling** (shrink hand motion) and **tremor filtering** (remove involuntary shake).

## Key facts & formulas

- **Master-slave loop**: master pose → mapping/transform → slave command; optionally slave force → master ([[Force Feedback & Haptics|haptics]]). Same shape as the [[VR Teleop Pipeline|FENCE-BOT pipeline]] (VR master → IK → sim slave).
- **Motion scaling**: slave moves a _fraction_ of master motion (e.g. 3:1 or 5:1 — surgeon moves 3 mm, instrument moves 1 mm). Turns coarse human motion into fine instrument motion — essential for delicate tissue work. A simple gain in the mapping, big clinical effect.
- **Tremor filtering**: low-pass / notch out the ~8–12 Hz physiological hand tremor before commanding the slave. A filtering stage in the teleop pipeline — relates to [[Complementary Filter|sensor filtering]] and adds phase lag you must budget for ([[Frequency-Domain Analysis|phase margin]], [[Latency Budgets]]).
- **Clutching**: decouple master and slave to re-center the master without moving the instrument (like lifting a mouse) — needed because the master workspace is smaller than the task.
- **Latency is safety-critical**: the master-slave loop must be low-latency and deterministic, or the surgeon feels lag/instability — which is _why_ the [[iceoryx Zero-Copy|zero-copy]] / [[Real-Time Determinism|real-time]] work matters here specifically.


## Links

- Related: [[Multi-Arm Laparoscopy]], [[VR Teleop Pipeline]], [[Force Feedback & Haptics]], [[Latency Budgets]], [[Virtual Fixtures]], [[iceoryx Zero-Copy]]
- Parent: [[00 Knowledge Map]]

---

