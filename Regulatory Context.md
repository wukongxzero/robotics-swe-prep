---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, regulatory, iec62304, fda, domain-knowledge]

# Regulatory Context

> [!note] Domain knowledge — not your authorship, but know enough to be credible You worked _inside_ a regulated environment at Articulus, so you absorbed how it shapes engineering — but you didn't own the regulatory submissions. Frame as: "I understand how regulation constrains how we build, and I've worked under it," not "I did the regulatory work." For an RSE role this is exactly the right amount: enough to be a credible teammate, not pretending to be a regulatory affairs specialist.


---

## Core idea

Medical device software is built under standards and regulatory oversight that dictate not just _what_ the software does but _how_ it's developed, documented, traced, and verified. The point isn't bureaucracy for its own sake — it's a paper trail proving the system is safe and that every requirement was deliberately met and tested. For an engineer, it means discipline: requirements traceability, documented design, rigorous testing, change control.

## Key facts

### IEC 62304 — medical device software lifecycle

- The international standard for **medical device software life-cycle processes**. Governs how you plan, develop, maintain, and manage risk in software — not the algorithms, the _process_.
- **Software safety classification** (drives rigor):
    - **Class A**: no injury possible.
    - **Class B**: non-serious injury possible.
    - **Class C**: death or serious injury possible. ← a surgical robot's control software.
    - Higher class = more required documentation, testing, and traceability.
- Demands: requirements → architecture → detailed design → implementation, each **traceable**, with **risk management** (tied to ISO 14971) and verification at each step.

### FDA pathways (US)

- **510(k)**: clearance via "substantial equivalence" to an existing legally-marketed device (predicate). Faster, most devices.
- **PMA (Premarket Approval)**: the stringent path for high-risk (Class III) devices — clinical evidence of safety/efficacy. Slower, more evidence.
- **De Novo**: for novel low/moderate-risk devices with no predicate.
- (Europe: **MDR** — Medical Device Regulation — the analogous framework.)

### Verification vs validation (regulators live on this distinction)

- **Verification**: "did we build the thing _right_?" — does it meet the specified requirements (testing against spec).
- **Validation**: "did we build the _right_ thing?" — does it meet the user's actual clinical need.
- Both required and documented; conflating them is a classic error.

### How it shapes engineering (the part that matters for me)

- **Requirements traceability**: every line of safety-relevant code traces to a requirement and a test. ([[Safety-Critical Architecture]])
- **Change control & documented design**: you can't just refactor the safety path on a whim — changes are controlled, risk-assessed, re-verified.
- **Risk management (ISO 14971)**: hazard analysis drives design — identify what can go wrong, mitigate, prove the mitigation.
- This is _why_ safety-critical architecture is built the way it is — the regulatory process and the technical safety case are two sides of one coin.

## Where it connects (Articulus)

- I built Class-C-level control software _inside_ this regulated environment — so I understand requirements traceability, documented design, change control, and the risk-management mindset from having worked under them. I did not own the regulatory submissions; my contribution was building software to that standard of discipline. That's the honest, and sufficient, framing for an RSE role.


## Links

- Related: [[Safety-Critical Architecture]], [[Multi-Arm Laparoscopy]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

