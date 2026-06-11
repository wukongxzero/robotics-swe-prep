---

## type: concept domain: Control theory status: drafted last-reviewed: tags: [control, sensor-fusion, imu, complementary-filter]

# Complementary Filter


---

## Core idea

A complementary filter fuses two sensors with **opposite error characteristics** by high-pass filtering one and low-pass filtering the other so their weaknesses cancel. The classic case: a gyro (accurate short-term, but _drifts_) and an accelerometer (noisy short-term, but _stable_ long-term as a gravity reference) → a clean attitude estimate.

## Key facts & formulas

- **The fusion** (per timestep, for angle $\theta$): $$\theta_k = \alpha,(\theta_{k-1} + \omega_{gyro},dt) ;+; (1-\alpha),\theta_{accel}$$
- $\alpha$ near 1 (e.g. 0.98): trust the integrated gyro short-term (high-pass), let a small slice of the accelerometer correct drift long-term (low-pass).
- **Why it works**: gyro integration drifts slowly (low-frequency error) → high-pass it. Accelerometer is noisy and corrupted by linear acceleration (high-frequency error) → low-pass it. The two filters are _complementary_ — they sum to 1 across frequency.
- **α and the time constant**: $\tau = \frac{\alpha, dt}{1-\alpha}$ sets the crossover — how long before the accelerometer pulls the estimate back.
- Dirt-cheap: a few multiply-adds per axis, one tuning constant. No matrices, no covariance.


## Links

- Related: [[Kalman Filter]], [[LQG]], [[State-Space & Pole Placement]], [[Sensor Fusion]]
- Parent: [[00 Knowledge Map]]

---

