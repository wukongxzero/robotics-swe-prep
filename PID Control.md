---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-05-30 tags: [control, pid]

# PID Control


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


## Links

- Related: [[State-Space & Pole Placement]], [[LQR]], [[Complementary Filter]]
- Parent: [[00 Knowledge Map]]

---

