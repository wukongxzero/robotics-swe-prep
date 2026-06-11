---
type: concept
domain: Real-time & embedded
status: drafted
last-reviewed:
tags: [motor-control, FOC, BLDC, Clarke, Park, PWM, embedded]
---

# Motor Control

> [!question] Explain it cold
> *Answer from memory first.*
>
> - What is a BLDC motor and how does it differ from a brushed DC motor?
> - What problem does 6-step commutation have, and how does FOC fix it?
> - Walk through the FOC pipeline step by step.

---

## Core idea

Field-Oriented Control (FOC) is a technique for driving BLDC motors that keeps the stator's magnetic field **always perpendicular to the rotor's magnetic field** — the condition for maximum torque efficiency. It does this by continuously transforming 3-phase currents into a rotor-aligned reference frame where you can control torque and flux independently as if they were DC quantities.

## Key facts

### BLDC vs brushed DC
- **Brushed DC**: commutation is mechanical (brushes + commutator). Simple to drive (voltage = speed), but brushes wear, create noise, and limit speed.
- **BLDC**: commutation is electronic — you must energize the correct coils at the right rotor angle. Faster, more efficient, longer life, but requires position feedback and smarter control.

### 6-step commutation (the naive approach)
- Energize 2 of 3 phases in fixed patterns, commutating every 60°.
- Problem: the torque-producing force vector **jumps in 60° steps** relative to the rotor → torque ripple, audible noise, reduced efficiency.
- Fine for fans and pumps; not acceptable for precise robotic actuation.

### FOC pipeline (what you need to know cold)

```
Measure phase currents (Ia, Ib, Ic)
        ↓
   Clarke Transform
        ↓
  (Iα, Iβ) — stationary 2-axis frame
        ↓
 Park Transform (needs rotor angle θ from encoder)
        ↓
  (Id, Iq) — rotating rotor-aligned frame
        ↓
  Two PID loops:
    Id PID → Vd  (flux axis — set Id_ref = 0 for max torque/amp)
    Iq PID → Vq  (torque axis — Iq_ref = your torque command)
        ↓
 Inverse Park Transform
        ↓
  (Vα, Vβ)
        ↓
 Inverse Clarke Transform (or SVPWM)
        ↓
  3-phase PWM duty cycles → motor
```

### Clarke Transform
Converts 3-phase currents to a 2-axis **stationary** frame (α, β). Reduces 3 variables to 2 (the third is redundant since Ia+Ib+Ic=0).

$$I_\alpha = I_a$$
$$I_\beta = \frac{1}{\sqrt{3}}(I_a + 2I_b)$$

Result: still AC (sinusoidal), but 2D instead of 3D.

### Park Transform
Rotates the stationary (α, β) frame into a frame that **spins with the rotor** (d, q). Requires real-time rotor angle $\theta$ from an encoder or resolver.

$$I_d = I_\alpha \cos\theta + I_\beta \sin\theta$$
$$I_q = -I_\alpha \sin\theta + I_\beta \cos\theta$$

Result: in the rotor frame, sinusoidal currents become **DC quantities**. You can now run simple PID on them.

- **$I_d$** (direct axis) = flux-producing component. Set $I_{d,\text{ref}} = 0$ for maximum torque per amp efficiency.
- **$I_q$** (quadrature axis) = torque-producing component. This is your throttle — command it to control torque directly.

### Why this gives smooth torque
Because the control always operates in the rotor's frame, the applied force vector stays continuously aligned with the rotor's magnetic field — no 60° jumps. Torque is proportional to $I_q$ directly.

### H-bridge & PWM drive
- An H-bridge has 4 switches (transistors/MOSFETs) per phase. Switching patterns determine direction and magnitude of current.
- PWM duty cycle controls the average voltage applied. Higher duty = more current = more torque.
- FOC outputs a duty cycle per phase via inverse Clarke/SVPWM after the PID loops.
- **Dead time**: brief pause when switching high/low side to prevent shoot-through (both MOSFETs on = short circuit).

### Encoders & position feedback
- FOC requires real-time rotor angle for the Park transform — typically from a **magnetic encoder** (e.g., AS5047) or hall sensors.
- Hall sensors give 6 positions per electrical revolution (coarse, enough for 6-step, noisy for FOC).
- Magnetic encoders give 12–14 bit resolution (4096–16384 positions/rev) — smooth FOC.
- **Dynamixel** servos have FOC built in with position/velocity/current control modes; you communicate over TTL/RS-485 protocol and set control mode + goal.


## Interview follow-ups
- **Q:** Why set $I_d = 0$ in FOC?
  - **A:** $I_d$ produces magnetic flux but not torque — it heats the motor without doing useful work. Setting it to zero maximizes torque per amp (efficiency). Field weakening intentionally uses non-zero $I_d$ to extend speed range beyond base speed, at the cost of efficiency.
- **Q:** What does the Park transform need that Clarke doesn't?
  - **A:** The rotor's current electrical angle $\theta$ — from an encoder. Clarke is a fixed geometric transform; Park is a rotation that tracks the rotor in real time.
- **Q:** FOC vs 6-step — when would you use 6-step?
  - **A:** 6-step is simpler (no encoder required for hall-sensor commutation, no transforms) and sufficient for fans, pumps, and simple BLDC drives. FOC is worth the complexity when you need smooth torque, precise current control, or quiet operation — robotics actuators.
- **Q:** What is field weakening?
  - **A:** Running $I_d < 0$ intentionally to reduce air-gap flux, which allows higher RPM beyond base speed (back-EMF limit). Used in EVs and high-speed spindles. Trades torque/amp efficiency for extended speed range.


## Links
- Related: [[AVR Peripherals]], [[Real-Time Determinism]], [[PID Control]], [[Serial Packet Protocols]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does FOC stand for and what is its core goal?
?
Field-Oriented Control. Keeps the stator field perpendicular to the rotor field at all times for maximum torque efficiency.

What problem does 6-step commutation have?
?
The torque-producing force vector jumps in 60° increments relative to the rotor, causing torque ripple and noise.

What does the Clarke transform do?
?
Converts 3-phase currents (Ia, Ib, Ic) into a 2-axis stationary frame (Iα, Iβ). Reduces 3 variables to 2 since they sum to zero.

What does the Park transform do, and what does it need?
?
Rotates the stationary (α,β) frame into a rotor-aligned rotating frame (d,q). Needs the real-time rotor electrical angle θ from an encoder.

What are Id and Iq?
?
Id = flux-producing current (set to 0 for max efficiency). Iq = torque-producing current (your torque throttle).

Why is FOC smooth compared to 6-step?
?
The control always operates in the rotor's frame, so the applied force vector continuously tracks the rotor — no discrete 60° jumps.

What is field weakening?
?
Running Id < 0 to reduce air-gap flux, allowing higher RPM beyond base speed at the cost of torque/amp efficiency.

What is the electrical angle vs mechanical angle?
?
θ_electrical = p × θ_mechanical, where p is the number of pole pairs. Park transform needs electrical angle.
