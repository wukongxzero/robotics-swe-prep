---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, pid]

# PID Control

> [!question] Explain it cold
> 
> - What does each of the three terms do?
> - What's integral windup and how do you stop it?
> - Why does the derivative term cause trouble in practice?

---

## Core idea

PID drives an error $e(t) = r(t) - y(t)$ (setpoint minus measurement) to zero by summing three responses: proportional to the error now, the integral of accumulated error, and the derivative of how fast error is changing.

$$u(t) = K_p e(t) + K_i \int_0^t e(\tau),d\tau + K_d \frac{de(t)}{dt}$$

## Key facts & formulas

- **P** — push proportional to current error. Higher $K_p$ = faster response but more overshoot; alone it leaves steady-state error.
- **I** — accumulates past error to kill steady-state offset. Too much → oscillation and windup.
- **D** — reacts to error _rate_, adds damping, anticipates. Amplifies measurement noise.
- **Discrete form** (what you actually code at fixed dt): integral becomes a running sum × dt, derivative becomes $(e_k - e_{k-1})/dt$.
- **Integral windup**: if the actuator saturates, the integral keeps accumulating while the output can't respond → huge overshoot when it unsaturates. Fixes: clamp the integral, conditional integration (stop integrating when saturated), back-calculation.
- **Derivative kick**: a step change in setpoint makes $de/dt$ spike. Fix: take the derivative of the _measurement_ ($-dy/dt$) instead of the error, and low-pass filter it.


## Interview follow-ups

- **Q:** Why not just use PID for the gimbal instead of LQR?
    - **A:** PID tunes each loop in isolation and ignores coupling between states; for a multi-state system (angle + rate, coupled axes) LQR optimizes the whole state vector at once with a principled cost tradeoff. PID is fine for SISO; state-space shines when states interact.
- **Q:** Your integral term causes huge overshoot after the actuator was maxed out — what's happening?
    - **A:** Integral windup — the integrator kept accumulating during saturation. Clamp it or use conditional integration / back-calculation anti-windup.
- **Q:** Adding D made things worse — why?
    - **A:** Derivative amplifies measurement noise and causes derivative kick on setpoint steps. Filter the derivative and take it on the measurement, not the error.


## Links

- Related: [[State-Space & Pole Placement]], [[LQR]], [[Complementary Filter]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does each PID term contribute? ? P: response proportional to current error (speed, but leaves offset). I: accumulates past error to kill steady-state error. D: reacts to error rate for damping/anticipation (amplifies noise).

What is integral windup and one fix? ? The integrator keeps accumulating while the actuator is saturated, causing large overshoot on recovery. Fix: clamp the integral, conditional integration, or back-calculation.

Why take the derivative of the measurement instead of the error? ? A setpoint step makes the error derivative spike ("derivative kick"). Using −dy/dt (measurement) avoids the kick; filtering tames noise amplification.