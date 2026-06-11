---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, virtual-fixtures, safety, control]

# Virtual Fixtures

> [!note] Domain + research-direction The surgical-robotics framing of virtual fixtures; the control/MPC implementation lives in [[MPC & Virtual Fixtures]] (Kim project). This note is the _why it matters clinically_ companion.

> [!question] Explain it cold
> 
> - What is a virtual fixture, in surgical terms?
> - Guidance vs forbidden-region — the two types.
> - How is a virtual fixture different from just teleoperation?

---

## Core idea

A virtual fixture is software-enforced **active assistance/constraint** on a teleoperated instrument — a "virtual jig" that guides the tool along helpful paths or keeps it out of dangerous regions. It augments the surgeon's control rather than replacing it: the human still drives, but the software shapes what motions are possible.

## Key facts & formulas

- **Two types**:
    - **Guidance virtual fixture**: gently steers the tool toward/along a desired trajectory (e.g. anisotropic stiffness — easy to move along the path, resistance off it). Helps follow a plan precisely.
    - **Forbidden-region virtual fixture**: a hard constraint keeping the instrument _out_ of a protected volume (a "no-fly zone" around a critical structure — a nerve, vessel, organ). Pure safety.
- **Mechanism**: implemented as a constraint in the teleop control loop — modifying commanded motion, adding restoring forces ([[Force Feedback & Haptics|haptics]]), or as a state constraint in an optimizer.
- **Why MPC is the principled implementation**: a forbidden region is a state constraint $x \in \mathcal{X}_{safe}$; [[MPC & Virtual Fixtures|MPC]] enforces it _predictively_ over a horizon, so the tool slows/redirects _before_ reaching the boundary rather than reacting at it. Anticipation = safety.
- **vs plain teleop**: teleop just maps motion ([[Teleoperation & Motion Scaling]]); a virtual fixture _adds intelligence/constraint_ on top — the system actively prevents or assists, not just mirrors.
- **Safety tie-in**: forbidden-region fixtures are a software safety layer ([[Safety-Critical Architecture]]) — protecting critical anatomy even if the surgeon's input would otherwise cross it.

## Where it connects

- **Research direction (Kim project)**: MPC-based virtual fixtures for teleoperated manipulation, simulation-only, Spring 2027 — the implementation note is [[MPC & Virtual Fixtures]]. This is forward-looking, framed honestly as the planned project.
- **Articulus / teleop**: the natural safety augmentation on a master-slave system like the one I worked on — domain-level connection.

## Interview follow-ups

- **Q:** What's a virtual fixture and what are the two kinds?
    - **A:** Software-enforced active assistance on a teleoperated tool. Guidance fixtures steer the tool along a desired path (anisotropic stiffness); forbidden-region fixtures keep it out of a protected volume (a no-fly zone around critical anatomy). The surgeon still drives; the software shapes the motion.
- **Q:** How is that different from teleoperation?
    - **A:** Teleop just maps the operator's motion to the instrument. A virtual fixture adds constraint/assistance on top — actively preventing unsafe motion or guiding toward a plan, rather than passively mirroring.
- **Q:** Why implement a forbidden region with MPC?
    - **A:** It's a state constraint; MPC enforces constraints predictively over a horizon, so the tool decelerates/redirects before reaching the boundary instead of reacting at it. Anticipation is the safety advantage over a reactive barrier.


## Links

- Related: [[MPC & Virtual Fixtures]], [[Teleoperation & Motion Scaling]], [[Safety-Critical Architecture]], [[Force Feedback & Haptics]], [[Multi-Arm Laparoscopy]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is a virtual fixture and what are the two types? ? Software-enforced active assistance/constraint on a teleoperated tool. Guidance: steers along a desired path (anisotropic stiffness). Forbidden-region: keeps the tool out of a protected volume (no-fly zone). Surgeon still drives.

How does a virtual fixture differ from plain teleoperation? ? Teleop just maps operator motion to the instrument; a virtual fixture adds constraint/assistance on top — actively preventing unsafe motion or guiding toward a plan, not just mirroring.

Why implement a forbidden-region fixture with MPC? ? It's a state constraint x ∈ X_safe; MPC enforces it predictively over a horizon, so the tool decelerates/redirects before the boundary rather than reacting at it — anticipation is the safety win.