---

## type: concept domain: Surgical robotics status: drafted last-reviewed: tags: [surgical, regulatory, iec62304, fda, domain-knowledge]

# Regulatory Context

> [!note] Domain knowledge — not your authorship, but know enough to be credible You worked _inside_ a regulated environment at Articulus, so you absorbed how it shapes engineering — but you didn't own the regulatory submissions. Frame as: "I understand how regulation constrains how we build, and I've worked under it," not "I did the regulatory work." For an RSE role this is exactly the right amount: enough to be a credible teammate, not pretending to be a regulatory affairs specialist.

> [!question] Explain it cold
> 
> - What does IEC 62304 govern, and how does it change how you write software?
> - FDA pathways: what's the difference between 510(k) and PMA, roughly?
> - What's "verification vs validation" and why do regulators care?

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

## Interview follow-ups

- **Q:** Have you worked under medical device regulation?
    - **A:** Yes — at Articulus I built control software for a Class-C-level system under IEC 62304-style discipline: requirements traceability, documented design, change control, risk-driven design. I wasn't the regulatory affairs owner, but I understand how the standards shape how you architect, document, and verify safety-critical software.
- **Q:** What does IEC 62304 actually require of you as a developer?
    - **A:** A controlled, documented, traceable lifecycle scaled by safety class — requirements → design → implementation → verification, each traceable, with risk management throughout. For Class C (serious injury possible) the rigor is highest. It governs process and evidence, not the algorithms.
- **Q:** Verification vs validation?
    - **A:** Verification: did we build it right (meets spec)? Validation: did we build the right thing (meets the clinical need)? Both are required and documented; they're not interchangeable.
- **Q:** 510(k) vs PMA?
    - **A:** 510(k) is clearance by substantial equivalence to an existing predicate device — faster, most devices. PMA is the stringent approval path for high-risk Class III devices, requiring clinical evidence of safety and efficacy.


## Links

- Related: [[Safety-Critical Architecture]], [[Multi-Arm Laparoscopy]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does IEC 62304 govern, and what scales its rigor? ? The medical device software life-cycle process (plan, develop, maintain, manage risk) — not the algorithms. Rigor scales with safety class: A (no injury), B (non-serious), C (death/serious injury) — a surgical robot's control software is Class C.

Verification vs validation? ? Verification: did we build it right (meets spec)? Validation: did we build the right thing (meets the clinical need)? Both required and documented; not interchangeable.

510(k) vs PMA (FDA)? ? 510(k): clearance by substantial equivalence to an existing predicate device — faster, most devices. PMA: stringent approval for high-risk Class III devices, requiring clinical safety/efficacy evidence.

How does medical regulation shape day-to-day engineering? ? Requirements traceability (code → requirement → test), documented design, controlled/risk-assessed changes, and risk management (ISO 14971) driving design. The regulatory process and the technical safety case are two sides of one coin.

How should you frame your regulatory experience honestly? ? "I built Class-C-level software under IEC 62304-style discipline and understand how regulation shapes architecture, documentation, and verification" — worked under it, did not own the submissions.