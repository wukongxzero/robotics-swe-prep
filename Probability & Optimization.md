---

## type: concept domain: SWE & math foundations status: drafted last-reviewed: tags: [math, probability, optimization]

# Probability & Optimization


---

## Core idea

Two math toolkits underpinning estimation and control. **Probability** models uncertainty (sensor noise, state belief) — the basis of filtering. **Optimization** finds the best decision under constraints — the basis of modern control and learning. Most robotics estimation is "probability + optimization" fused.

## Probability

- **Gaussian / normal** ($\mathcal N(\mu,\Sigma)$): the default because of the central limit theorem (sums of noise tend Gaussian) and because it's closed under linear operations — a Gaussian through a linear system stays Gaussian. Defined by **mean** $\mu$ and **covariance** $\Sigma$. This is _why_ the [[Kalman Filter]] is clean: linear-Gaussian in, Gaussian out.
- **Covariance** $\Sigma$: uncertainty + correlation between variables; PSD. Carried as the state-uncertainty in [[Kalman Filter|Kalman]] and [[Sensor Fusion|fusion]].
- **Bayes' rule**: posterior ∝ likelihood × prior — the backbone of estimation (the Kalman update is Bayes for the linear-Gaussian case).
- **Maximum likelihood / MAP**: estimate parameters by maximizing data probability — connects to least-squares (Gaussian MLE = least squares).

## Optimization

- **Convex vs non-convex**: convex problems have a _single global_ optimum, solvable reliably and fast. Non-convex can have local minima — no global guarantee. Recognizing/convexifying a problem is huge.
- **QP (Quadratic Program)**: quadratic cost + linear constraints — convex, fast, well-solved. **In robotics**: [[MPC & Virtual Fixtures|MPC]] reduces to a QP each step; [[Floating-Base & Whole-Body|whole-body control]] is a hierarchical QP. The workhorse.
- **NLP (Nonlinear Program)**: general nonlinear — [[Trajectory Optimization|trajopt]], nonlinear MPC. Solved with SQP/interior-point (IPOPT), needs gradients ([[acados OCS2 CasADi|CasADi autodiff]]).
- **Gradient descent**: first-order, for large/unstructured problems (ML training). **Newton / second-order**: uses curvature (Hessian), faster convergence near the optimum, costlier per step.
- **Least squares**: minimize $|Ax-b|^2$ — closed-form via pseudoinverse ([[Linear Algebra Refresher]]); the link between probability (Gaussian MLE) and optimization.

## Where it connects

- **Estimation**: [[Kalman Filter]], [[Sensor Fusion]], SLAM back-end ([[SLAM & RTAB-Map]]) are all probability + least-squares optimization.
- **Control**: [[LQR]] (quadratic cost), [[MPC & Virtual Fixtures|MPC]] (QP), [[Trajectory Optimization]] (NLP).
- **ML**: [[Lagrangian Neural Networks|LNN]]/[[PINN Contact Estimation|PINN]] training is gradient-based non-convex optimization.


## Links

- Related: [[Kalman Filter]], [[Sensor Fusion]], [[LQR]], [[MPC & Virtual Fixtures]], [[Trajectory Optimization]], [[Linear Algebra Refresher]], [[acados OCS2 CasADi]]
- Parent: [[00 Knowledge Map]]

---

