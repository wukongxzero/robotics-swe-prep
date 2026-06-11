---
type: concept
domain: Perception & ML
status: drafted
last-reviewed:
tags: [sensor-fusion, kalman, imu, estimation, complementary-filter]
---

# Sensor Fusion


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


## Links
- Related: [[Kalman Filter]], [[Complementary Filter]], [[SLAM & RTAB-Map]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

