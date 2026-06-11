---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, virtual-fixtures, safety, control]

# Virtual Fixtures

> [!note] Domain + research-direction The surgical-robotics framing of virtual fixtures; the control/MPC implementation lives in [[MPC & Virtual Fixtures]] (Kim project). This note is the _why it matters clinically_ companion.


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


## Links

- Related: [[MPC & Virtual Fixtures]], [[Teleoperation & Motion Scaling]], [[Safety-Critical Architecture]], [[Force Feedback & Haptics]], [[Multi-Arm Laparoscopy]]
- Parent: [[00 Knowledge Map]]

---

