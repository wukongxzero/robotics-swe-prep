---

## type: concept domain: Control theory status: drilled last-reviewed: 2026-06-08 tags: [control, kalman, estimation, sensor-fusion, deep]

# Kalman Filter

> [!star] Interviewer will push here — full predict/update below The Kalman filter is the "I fuse noisy sensors into a clean state estimate" story. Know the two-step cycle, the gain, and what the covariances mean, cold.

> [!question] Explain it cold
> 
> - What problem does the Kalman filter solve?
> - Write the predict and update steps.
> - What does the Kalman gain actually balance? What do Q and R mean here?

---

## Core idea

The Kalman filter is the **optimal recursive estimator** for a linear system with Gaussian noise. It fuses a _model prediction_ with a _noisy measurement_, weighting each by its uncertainty, to produce a state estimate with minimum variance. It carries not just an estimate $\hat{x}$ but its uncertainty $P$ (covariance), and updates both every step.

## Model

$$x_k = A x_{k-1} + B u_{k-1} + w_{k-1}, \quad w \sim \mathcal{N}(0, Q)$$ $$z_k = H x_k + v_k, \quad v \sim \mathcal{N}(0, R)$$

- $Q$ = **process noise** covariance (how much I trust the model). Bigger Q → trust measurements more.
- $R$ = **measurement noise** covariance (how noisy the sensor is). Bigger R → trust the model more.
- ⚠️ These Q, R are _not_ the LQR Q, R — same letters, different objects.

## The two-step cycle

**Predict (time update)** — push state and uncertainty forward through the model: $$\hat{x}_k^- = A\hat{x}_{k-1} + Bu_{k-1}$$ $$P_k^- = A P_{k-1} A^T + Q$$ (uncertainty _grows_ — the model isn't perfect.)

**Update (measurement update)** — correct with the new measurement: $$\hat{x}_k = \hat{x}_k^- + K_k\left(z_k - H\hat{x}_k^-\right) \quad \text{(innovation correction)}$$ $$P_k = (I - K_k H)P_k^- \quad \text{(uncertainty shrinks)}$$

## What the gain means (the intuition)

- $z_k - H\hat{x}_k^-$ is the **innovation** — how surprised the measurement makes us.
- $K_k$ scales how much we trust that surprise. If sensor noise $R$ is large, $K \to 0$ (ignore measurement, trust model). If prediction uncertainty $P^-$ is large, $K \to$ large (trust measurement, override model). The gain is the optimal blend of the two uncertainties.

## Variants

| Filter | When to use | Key difference |
|--------|-------------|----------------|
| **Linear KF** | Linear dynamics + linear sensor | Exact — no approximation |
| **EKF** | Nonlinear dynamics or sensor | Jacobian linearizes f(x) at each step |
| **UKF** | Strongly nonlinear | Sigma points propagated through f(x) — no Jacobian |

### EKF — Extended Kalman Filter

Real robots are nonlinear. Example: pendulum has $\ddot{\theta} = -(g/L)\sin\theta$ — the $\sin$ breaks linearity.

**Key idea:** at each step, linearize $f(x)$ around the current estimate using its Jacobian $F = \partial f / \partial x$.

**EKF predict:**
$$\hat{x}^- = f(\hat{x}) \qquad \text{(propagate through the nonlinear function)}$$
$$P^- = F P F^T + Q \qquad \text{(propagate covariance through the Jacobian)}$$

**EKF update:** same as linear KF — if the sensor is linear, H is constant.

**Jacobian for pendulum** (changes every step because $\cos\theta$ changes):
$$F = \begin{bmatrix} 1 & dt \\ -(g/L)\cos\theta \cdot dt & 1 \end{bmatrix}$$

**Interview one-liner:** "EKF linearizes the nonlinear motion model at the current estimate via the Jacobian each timestep — the workhorse for SLAM and pose estimation."

### UKF — Unscented Kalman Filter

**Key idea:** instead of linearizing, pick $2n+1$ **sigma points** around the current estimate, push each through $f(x)$ directly, then reconstruct the mean and covariance from the results. No Jacobian derivation needed.

**Sigma points** (n = state dimension, $\lambda$ = scaling parameter):
$$\chi_0 = \hat{x}, \quad \chi_i = \hat{x} + (\sqrt{(n+\lambda)P})_i, \quad \chi_{i+n} = \hat{x} - (\sqrt{(n+\lambda)P})_i$$

**UKF predict:**
1. Generate $2n+1$ sigma points
2. Push each through $f(x)$ — no linearization
3. Reconstruct $\hat{x}^-$ and $P^-$ as weighted sum of propagated points

**UKF vs EKF tradeoff:**
- UKF is more accurate for strongly nonlinear systems (captures higher-order effects)
- UKF requires $2n+1$ function evaluations per step vs 1 for EKF
- UKF avoids deriving Jacobians (saves implementation effort, avoids derivation errors)

**Interview one-liner:** "UKF avoids Jacobians by propagating sigma points through the nonlinear function — better accuracy for strong nonlinearity at the cost of more function evaluations."

## Implementations (Robot_practice)

- `kalman_filter.cpp` — working C++ implementation with Eigen (built 2026-06-08)
- `kalman_filter_annotated.cpp` — same code, every line explained
- `kalman_variants.py` — all three variants (Linear KF, EKF, UKF) with comparative plots
  - Linear KF: 1D position/velocity tracking
  - EKF: pendulum with sin(θ) nonlinearity, Jacobian recomputed each step
  - UKF: same pendulum, sigma point propagation, no Jacobian

All in `/home/wukong/Robot_practice/core_robotics_programs/`

## Where I've used it

- **WALL-E V3 gimbal**: ran a **discrete Kalman filter** in the [[LQG]] variant for attitude estimation at 200 Hz on the Arduino UNO — fusing IMU readings into angle+rate state. The [[Complementary Filter]] was the lightweight alternative I also implemented; the Kalman version is the principled one when you want to model the noise explicitly.

## Interview follow-ups

- **Q:** What does the Kalman gain balance?
    - **A:** Prediction uncertainty ($P^-$) against measurement noise ($R$). High sensor noise → gain near zero, trust the model; high prediction uncertainty → high gain, trust the measurement. It's the minimum-variance blend.
- **Q:** Walk me through the two steps.
    - **A:** Predict: propagate state through A (plus input B·u) and grow covariance by Q. Update: compute gain from $P^-$ and R, correct the state by gain × innovation, shrink covariance by $(I-KH)$.
- **Q:** What if the system is nonlinear?
    - **A:** EKF — linearize about the current estimate with Jacobians each step. UKF if the nonlinearity is strong, using sigma points to avoid linearization error.
- **Q:** Kalman vs complementary filter for the gimbal — why pick one?
    - **A:** Complementary filter is cheap, tuning-by-one-constant, great when I just need to fuse a fast-but-drifting gyro with a slow-but-stable accelerometer. Kalman is principled, models noise covariances, optimal under Gaussian assumptions, and extends to more states — at higher compute and tuning cost. On an ATmega at 200 Hz that tradeoff is real.

## Computational cost & the inversion problem

The update step requires inverting $(HP^-H^T + R)$ — an $m \times m$ matrix where $m$ = number of measurements. Cost is $O(m^3)$.

- **1 sensor** → scalar divide, trivial
- **Many sensors** → expensive and numerically unstable

**Interview question:** "The Kalman filter has more inversions than you'd think — what's the cost and how do you handle it?"

**Answer:**
- **Joseph form** — numerically stable version of the covariance update: $P = (I-KH)P(I-KH)^T + KRK^T$ instead of $(I-KH)P$. Avoids asymmetry from floating point errors.
- **Square root filter** — propagate $\sqrt{P}$ (Cholesky factor) instead of $P$ directly. More stable, never lets $P$ go non-positive-definite.
- **Information filter** — work in inverse covariance space $(\Omega = P^{-1})$. Avoids the inversion for update-heavy scenarios.
- **Sequential scalar updates** — split a large measurement vector into individual scalar updates. Each update is a scalar divide — no matrix inversion at all.

## Gotchas / what trips me up

- Confusing Kalman's Q/R (process/measurement noise) with LQR's Q/R (state/effort weights). Different objects, same letters.
- Forgetting covariance _grows_ in predict and _shrinks_ in update.
- Assuming linearity — real robots usually need EKF.

## Links

- Related: [[LQG]], [[Complementary Filter]], [[State-Space & Pole Placement]], [[Sensor Fusion]], [[SLAM & RTAB-Map]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does the Kalman filter optimally fuse, and what does it track besides the estimate? ? A model prediction and a noisy measurement, weighted by their uncertainties (minimum-variance under Gaussian noise). It also tracks the estimate's covariance P.

The two predict-step equations? ? x̂⁻ = Ax̂ + Bu (propagate state); P⁻ = APAᵀ + Q (grow covariance by process noise Q).

Kalman gain formula and what it balances? ? K = P⁻Hᵀ(HP⁻Hᵀ + R)⁻¹. It balances prediction uncertainty P⁻ against measurement noise R — high R → trust model, high P⁻ → trust measurement.

In the Kalman filter, what are Q and R (vs LQR)? ? Here Q = process noise covariance (model trust), R = measurement noise covariance (sensor trust) — NOT LQR's state/effort weights, despite the same letters.

EKF vs UKF? ? EKF linearizes the nonlinear model with Jacobians each step; UKF propagates sigma points through the nonlinearity (better for strong nonlinearity, no Jacobians).

EKF predict step — what's different from linear KF? ? State: x = f(x) — propagate through the NONLINEAR function directly. Covariance: P = FPFᵀ + Q — propagate through the JACOBIAN F = ∂f/∂x evaluated at current x. Linear KF uses A everywhere; EKF uses f(x) for state and F (Jacobian) for covariance.

What are UKF sigma points and why 2n+1? ? Carefully chosen sample points around the current estimate: 1 center point + n points in each ± direction of the covariance square root. 2n+1 total captures the mean and spread without Jacobians. Each is pushed through f(x) directly.

When do you pick UKF over EKF? ? When the nonlinearity is strong enough that first-order linearization (EKF) introduces significant error — e.g., large angle swings, highly curved trajectories. UKF captures higher-order effects at the cost of 2n+1 function evaluations per step instead of 1.

What is S (innovation covariance) in the update step? ? S = H·P⁻·Hᵀ + R — the total expected variance of the measurement surprise. H·P⁻·Hᵀ is uncertainty projected into sensor space; +R adds sensor noise. K is normalized by S.

What is the computational cost of the Kalman update inversion, and what are the fixes? ? Inverting (HP⁻Hᵀ+R) costs O(m³) where m = number of measurements. Fixes: Joseph form (numerical stability), square root filter (propagate √P), information filter (inverse covariance space), or sequential scalar updates (avoid matrix inversion entirely).