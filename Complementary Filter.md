---

## type: concept domain: Control theory status: drafted last-reviewed: tags: [control, sensor-fusion, imu, complementary-filter]

# Complementary Filter

> [!question] Explain it cold
> 
> - What two sensors does it fuse for attitude, and what's wrong with each alone?
> - Write the one-line update and explain the α.
> - Why use it over a Kalman filter?

---

## Core idea

A complementary filter fuses two sensors with **opposite error characteristics** by high-pass filtering one and low-pass filtering the other so their weaknesses cancel. The classic case: a gyro (accurate short-term, but _drifts_) and an accelerometer (noisy short-term, but _stable_ long-term as a gravity reference) → a clean attitude estimate.

## Key facts & formulas

- **The fusion** (per timestep, for angle $\theta$): $$\theta_k = \alpha,(\theta_{k-1} + \omega_{gyro},dt) ;+; (1-\alpha),\theta_{accel}$$
- $\alpha$ near 1 (e.g. 0.98): trust the integrated gyro short-term (high-pass), let a small slice of the accelerometer correct drift long-term (low-pass).
- **Why it works**: gyro integration drifts slowly (low-frequency error) → high-pass it. Accelerometer is noisy and corrupted by linear acceleration (high-frequency error) → low-pass it. The two filters are _complementary_ — they sum to 1 across frequency.
- **α and the time constant**: $\tau = \frac{\alpha, dt}{1-\alpha}$ sets the crossover — how long before the accelerometer pulls the estimate back.
- Dirt-cheap: a few multiply-adds per axis, one tuning constant. No matrices, no covariance.


## Interview follow-ups

- **Q:** Why fuse gyro and accelerometer at all?
    - **A:** Gyro integrates to angle accurately short-term but drifts; accelerometer gives an absolute gravity reference that's stable long-term but noisy and fooled by linear acceleration. Each covers the other's failure band.
- **Q:** Complementary vs Kalman — when do you reach for which?
    - **A:** Complementary when I want minimal compute and one tuning knob and the noise is well-behaved — perfect for an MCU at high rate. Kalman when I want to model noise covariances explicitly, fuse more states/sensors, or need optimality. On the ATmega gimbal the complementary filter was the pragmatic default; the Kalman was the rigorous comparison.
- **Q:** What does α control?
    - **A:** The crossover frequency / time constant — how much to trust the integrated gyro vs how fast the accelerometer corrects drift. Higher α = trust gyro longer.


## Links

- Related: [[Kalman Filter]], [[LQG]], [[State-Space & Pole Placement]], [[Sensor Fusion]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What two sensors does a complementary filter fuse, and why? ? Gyro (accurate short-term but drifts → high-pass) and accelerometer (noisy short-term but stable gravity reference → low-pass). Their error bands are complementary.

The complementary filter update equation? ? θ = α(θ_prev + ω_gyro·dt) + (1−α)θ_accel. α near 1 trusts the integrated gyro short-term while the accelerometer slowly corrects drift.

Complementary filter vs Kalman — the tradeoff? ? Complementary: minimal compute, one tuning constant, no covariance — great for an MCU. Kalman: models noise explicitly, optimal, scales to more states — higher compute/tuning cost.