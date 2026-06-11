---

## type: concept domain: SWE & math foundations status: drafted last-reviewed: tags: [math, probability, optimization]

# Probability & Optimization

> [!question] Explain it cold
> 
> - Why is a Gaussian the default noise model, and what two parameters define it?
> - Convex vs non-convex — why do you care?
> - Gradient descent vs a QP solver — when each?

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

## Interview follow-ups

- **Q:** Why model noise as Gaussian?
    - **A:** Central limit theorem (aggregated noise tends Gaussian) and closure under linear operations — a Gaussian through a linear system stays Gaussian, defined by just mean and covariance. That's what makes the Kalman filter tractable.
- **Q:** Why care if a problem is convex?
    - **A:** Convex problems have a single global optimum and are solved reliably and fast; non-convex ones can trap solvers in local minima with no global guarantee. Formulating control as a convex QP (like MPC) is what makes it real-time solvable.
- **Q:** When QP vs general NLP?
    - **A:** QP (quadratic cost, linear constraints) is convex and fast — linear MPC and whole-body control use it. NLP handles nonlinear dynamics/costs (nonlinear MPC, trajopt) via SQP/interior-point, slower and needing gradients.

## Gotchas / what trips me up

- Conflating covariance (spread+correlation) with variance (single-variable spread).
- Assuming a solver found the global optimum on a non-convex problem.
- Forgetting Gaussian MLE = least squares (the probability↔optimization bridge).

## Links

- Related: [[Kalman Filter]], [[Sensor Fusion]], [[LQR]], [[MPC & Virtual Fixtures]], [[Trajectory Optimization]], [[Linear Algebra Refresher]], [[acados OCS2 CasADi]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Why is the Gaussian the default noise model? ? Central limit theorem (aggregated noise tends Gaussian) + closure under linear operations (Gaussian through a linear system stays Gaussian, defined by mean and covariance) — which makes the Kalman filter tractable.

Convex vs non-convex — why does it matter? ? Convex problems have a single global optimum, solved reliably and fast; non-convex can trap solvers in local minima with no global guarantee. Formulating control as a convex QP makes it real-time solvable.

QP vs NLP in robotics? ? QP (quadratic cost + linear constraints, convex, fast): linear MPC, whole-body control. NLP (nonlinear): nonlinear MPC, trajectory optimization — via SQP/interior-point, needs gradients.