---

## type: concept domain: Mechanical & Design status: drafted last-reviewed: tags: [baja, solidworks, dfmea, mechanical, design]

# SAE BAJA & Engineering Design

> [!question] Explain it cold
>
> - What is SAE BAJA and what did you contribute?
> - Walk through a DFMEA — what does it catch that design review doesn't?
> - How do you go from SolidWorks assembly to a validated design?

---

## Core idea

SAE BAJA is a collegiate engineering competition where teams design, build, and race a single-seat off-road vehicle from scratch. Every system (suspension, drivetrain, brakes, frame) is student-designed and must pass technical inspection before racing. My role was **SolidWorks design** and **DFMEA** — systematic risk analysis to catch failure modes before they reach the build stage.

---

## SolidWorks design workflow

```
Concept sketch
    ↓
Part modeling (individual components)
    ↓
Assembly (mates, motion studies)
    ↓
FEA (stress/deflection under load cases)
    ↓
Drawing package (GD&T, tolerances, BOM)
    ↓
DFMEA review
    ↓
Design freeze → fabrication
```

**Key SolidWorks tools used:**
| Tool | Purpose |
|------|---------|
| Part/Assembly modeling | Full vehicle geometry, interference check |
| Motion Study | Suspension travel, steering geometry validation |
| Simulation (FEA) | Stress analysis under worst-case loads (bump, cornering) |
| Drawing package | Manufacturing drawings with GD&T |
| SolidWorks Toolbox | Standard fasteners, bearings from library |
| URDF Exporter | Robot arm export (separate from BAJA) |

**Suspension design specifics:**
- Double wishbone geometry → optimized camber curve, caster, KPI
- Motion ratio for spring/damper selection
- Ackermann steering geometry — inside wheel turns sharper than outside
- Static + dynamic load cases: bump (3G), cornering (1.5G lateral), braking

---

## DFMEA — Design Failure Mode and Effects Analysis

**What it is:** A structured, bottom-up risk analysis that asks for every component: *what can fail, how likely is it, how severe is the effect, how detectable is it?*

**DFMEA table structure:**

| Component | Function | Failure Mode | Effect | Severity (S) | Cause | Occurrence (O) | Detection (D) | RPN | Action |
|-----------|----------|-------------|--------|-------------|-------|----------------|---------------|-----|--------|
| Tie rod | Transmit steering force | Buckling under lateral load | Loss of steering | 9 | Undersized cross-section | 3 | 4 | 108 | Increase diameter, add FEA case |
| Wheel bearing | Support wheel radial/axial loads | Spalling/seizure | Wheel detachment | 10 | Over-specification of preload | 2 | 5 | 100 | Torque spec on assembly, inspection |

**RPN = Severity × Occurrence × Detection** (each 1–10)

- **Severity**: How bad is the effect? (10 = injury/death, 1 = no effect)
- **Occurrence**: How likely is the failure? (10 = almost certain, 1 = almost impossible)
- **Detection**: How likely to catch before it causes harm? (10 = undetectable, 1 = always detected)

**High RPN → priority action item.** Target: reduce via design change (lower S/O) or add detection (lower D).

**Why DFMEA catches what design review misses:**
- Forces you to enumerate *every* failure mode, not just obvious ones
- Quantifies risk → objective priority ranking
- Documents rationale for design decisions (traceable)
- Catches failure modes that only appear under specific load combinations

---

## BAJA vehicle systems overview

```
Frame (chromoly steel tube)
├── Suspension
│   ├── Front: double wishbone, push/pull rod
│   └── Rear: trailing arm or 4-link
├── Drivetrain
│   ├── Briggs & Stratton 10hp engine (spec engine)
│   ├── CVT (continuously variable transmission)
│   └── Gearbox + differential
├── Steering
│   ├── Rack and pinion
│   └── Ackermann geometry tie rods
├── Brakes
│   ├── Hydraulic disc (all 4 wheels)
│   └── Inboard rear brake
└── Ergonomics
    ├── Roll cage (ROPS) — must pass SAE spec
    └── Harness, firewall
```

---

## Key engineering principles from BAJA

**Ackermann steering geometry:**
```
For no tire scrub during cornering:
  Inner wheel angle > outer wheel angle
  tan(inner) - tan(outer) = track_width / wheelbase
  (true Ackermann — rarely achieved perfectly, tuned per vehicle)
```

**Spring rate selection:**
```
Natural frequency target: 1.5–2 Hz for off-road
  fn = (1/2π) × sqrt(k_wheel / m_corner)
  k_wheel = k_spring × (motion_ratio)²
  
Motion ratio = spring displacement / wheel displacement
```

**Factor of safety (FEA):**
- BAJA rule: FOS ≥ 1.5 on structural members under worst-case loads
- Yield strength not ultimate — first sign of permanent deformation
- Chromoly 4130 common: Sy = 435 MPa, Su = 670 MPa

---

## SolidWorks → URDF for robot arms (crossover skill)

This is where BAJA CAD skills transferred to robotics:
```
SolidWorks Assembly
    ↓ sw2urdf plugin (or manual export)
    ↓ STL meshes + joint definitions
URDF
    ↓
ROS2 / Isaac Lab / Gazebo
```
See [[URDF & CAD Pipeline]] for full workflow.

---

## Interview angle

BAJA demonstrates:
- End-to-end mechanical design ownership (concept → fabrication)
- Systems thinking (every subsystem affects others — suspension affects handling affects driver safety)
- Structured risk analysis (DFMEA) — directly relevant to safety-critical robotics
- CAD proficiency at assembly scale
- Working under real constraints (weight, cost, rules)

---

## Links

- Related: [[URDF & CAD Pipeline]], [[Safety-Critical Architecture]], [[Contact Modeling]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is RPN in DFMEA and how do you use it? ? RPN = Severity × Occurrence × Detection (each 1–10). High RPN = priority action. Reduce by design change (lower S or O) or adding detection mechanism (lower D). Forces quantified, traceable risk decisions.

Why does DFMEA catch things design review misses? ? It forces enumeration of every failure mode systematically, not just the obvious ones. Quantifies risk with RPN for objective prioritization. Catches failure modes that only appear under specific load combinations or manufacturing variation.

What is Ackermann steering geometry? ? For cornering without tire scrub, the inner wheel must turn sharper than the outer. True Ackermann: tan(inner) - tan(outer) = track_width / wheelbase. Ensures all four wheels rotate about a common instantaneous center.
