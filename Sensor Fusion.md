---
type: concept
domain: Perception & ML
status: drafted
last-reviewed:
tags: [sensor-fusion, kalman, imu, estimation, complementary-filter]
---

# Sensor Fusion

> [!question] Explain it cold
> *Answer from memory first.*
>
> - What is sensor fusion and why do you need it?
> - Name three fusion architectures and when you'd pick each.
> - What's the difference between a complementary filter and a Kalman filter for attitude estimation?

---

## Core idea

Sensor fusion combines measurements from multiple sensors to produce a state estimate **better than any single sensor alone**. Each sensor has different noise characteristics, biases, and update rates. Fusion exploits complementary strengths: IMUs are fast but drift; encoders slip; cameras are accurate but slow and compute-heavy. The fused estimate gets the best of each.

## Key facts & formulas

### Why individual sensors fail alone

| Sensor | Strength | Weakness |
|--------|----------|----------|
| IMU (accel + gyro) | High rate, no slip | Integrates noise → drift over time |
| Encoder | Precise incremental | Wheel slip, no absolute reference |
| Camera/vision | Rich, absolute features | Slow, sensitive to lighting, compute heavy |
| GPS | Absolute position | Outdoor only, noisy, 1–10 Hz |

### Fusion architectures

**Complementary filter (attitude)**
- Splits the frequency spectrum: gyro (good at high freq, bad at DC drift) + accelerometer (good at DC/low freq, bad at vibration).
- High-pass filter on gyro + low-pass filter on accel. The two filters sum to 1.
- Formula: $\hat{\theta}_k = \alpha(\hat{\theta}_{k-1} + \omega_k \Delta t) + (1-\alpha)\theta_{\text{accel},k}$
- $\alpha$ (0.95–0.98 typical): trust gyro more short-term, drift corrected by accel long-term.
- Cheap, no matrix math, works well for attitude. See [[Complementary Filter]].

**Kalman filter (full state)**
- Optimal fusion for linear Gaussian systems — propagate state + covariance through the model, then correct with measurements weighted by their uncertainty.
- Use when: you have a good dynamics model, multiple heterogeneous sensors at different rates, and need a principled uncertainty estimate.
- IMU + encoder + GPS fusion in ground robots. IMU + camera (VIO) for drones.
- See [[Kalman Filter]] for full predict/update equations.

**Extended Kalman Filter (EKF)**
- Nonlinear systems: linearize around current estimate via Jacobians. Used in SLAM, visual odometry.
- Works well when nonlinearity is mild. Diverges if linearization is too crude.

**Particle filter**
- Non-parametric, handles multimodal distributions. Used in SLAM (global localization — the robot doesn't know where it starts).
- Expensive (scales with particle count) but handles arbitrary uncertainty shapes.

### IMU + encoder fusion (odometry)
Common on mobile robots: integrate wheel odometry for position, fuse with IMU for heading correction.
- Encoder gives $\Delta x, \Delta y$ (but drifts from slip)
- IMU gyro gives $\dot{\theta}$ (but drifts from bias)
- Fused: EKF with state $[x, y, \theta]$, IMU as control input, encoder as measurement

### Visual-Inertial Odometry (VIO)
IMU + camera tightly coupled: IMU predicts at 200–1000 Hz, camera corrects at 30 Hz. Used in drones, AR, and legged robots where GPS is unavailable.

## Where I've used it
- **WALL-E V3 gimbal**: [[Complementary Filter]] for attitude — fused MPU-6050 gyro + accel to get stable pitch/roll. Chose complementary over Kalman because it's cheap (no matrix math) and sufficient for a 2-axis gimbal.
- **FENCE-BOT**: IMU + encoder fusion for dead-reckoning — encoder gives wheel displacement, IMU corrects heading. This is the `sensor_fusion` node in the stack.
- Conceptual foundation for [[SLAM & RTAB-Map]] — SLAM is sensor fusion at the map level.

## Interview follow-ups
- **Q:** Why use a complementary filter instead of a Kalman filter for attitude?
  - **A:** For attitude only, complementary is simpler (no matrices), computationally cheap, and gives equivalent performance. Kalman is worth the complexity when you have many heterogeneous sensors, need a formal uncertainty estimate, or are fusing more than 2 modalities.
- **Q:** What's the difference between loose, tight, and deep coupling in IMU+camera fusion?
  - **A:** Loose: fuse finished pose estimates. Tight: fuse raw features and IMU measurements together in one filter (more accurate, handles degeneracies better). Deep: integrate at the sensor hardware level (rare).
- **Q:** Your odometry drifts after 10 meters. What's causing it and what fixes it?
  - **A:** Encoder slip + IMU gyro bias accumulate. Fix: add loop closure (visual or LiDAR SLAM), absolute corrections (GPS, fiducial markers), or a better gyro. The drift is fundamental to dead-reckoning without absolute reference.

## Gotchas / what trips me up
- Complementary filter $\alpha$ too high → gyro drift accumulates; too low → accel noise bleeds through. Tune empirically.
- EKF can diverge if the initial covariance is badly set or the linearization assumption breaks down. When in doubt, particle filter is more robust.
- Sensor timestamps: fusing sensors at different rates requires careful interpolation — a 1 ms timestamp error in IMU data corrupts the prediction step.

## Links
- Related: [[Kalman Filter]], [[Complementary Filter]], [[SLAM & RTAB-Map]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is sensor fusion?
?
Combining measurements from multiple sensors to produce a state estimate better than any single sensor, exploiting their complementary strengths and weaknesses.

What does a complementary filter do for attitude estimation?
?
High-pass filters the gyro (good at fast changes, bad at DC drift) and low-pass filters the accelerometer (good at slow/static, noisy at vibration). They sum to 1, giving a drift-corrected attitude estimate.

What is the complementary filter formula?
?
θ_hat_k = α(θ_hat_{k-1} + ω_k·Δt) + (1-α)·θ_accel_k, where α ≈ 0.95–0.98.

When would you use Kalman over complementary filter?
?
When you have multiple heterogeneous sensors at different rates, need a formal uncertainty estimate, or have a good dynamics model. Complementary is sufficient for attitude-only with two sensors.

What is VIO?
?
Visual-Inertial Odometry — tight fusion of IMU (200–1000 Hz) and camera (30 Hz) for pose estimation without GPS. IMU predicts, camera corrects.

Why does dead-reckoning drift over time?
?
Encoder slip and IMU gyro bias accumulate with each integration step — errors compound. Needs absolute reference (loop closure, GPS, landmarks) to correct.
