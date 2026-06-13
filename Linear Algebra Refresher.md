---

## type: concept domain: SWE & math foundations status: drafted last-reviewed: tags: [math, linear-algebra]

# Linear Algebra Refresher


---

## Core idea

Linear algebra is the substrate under all of robotics — transforms, control, estimation, IK, optimization. The goal isn't to re-derive it, but to _recognize_ the same handful of objects (eigenvalues, SVD, pseudoinverse, positive-definiteness) wherever they recur, because they connect notes across the whole vault.

## Key facts & where each recurs

- **Eigenvalues/eigenvectors** ($Av = \lambda v$): directions a transform only scales. **In robotics**: eigenvalues of $(A-BK)$ are the closed-loop poles ([[State-Space & Pole Placement]]) — stability = all in the left-half plane. Eigenvalues of the inertia matrix = principal inertias.
- **SVD** ($A = U\Sigma V^T$): any matrix as rotation–scale–rotation; singular values measure "gain" per direction. **In robotics**: [[Jacobian]] singularities = a singular value → 0; manipulability uses singular values; numerical rank; PCA. The Swiss-army knife.
- **Pseudoinverse** ($A^+$): best least-squares solution to over/underdetermined $Ax=b$. **In robotics**: the basis of numerical [[Inverse Kinematics & DLS|IK]] (and DLS is the _damped_ pseudoinverse).
- **Positive-definiteness** ($x^TMx > 0$): "energy is always positive." **In robotics**: inertia matrix $M(q)$ is SPD ([[Robot Dynamics Formulations]]); [[LQR]] cost weights $Q\succeq0, R\succ0$; covariance matrices ([[Kalman Filter]]) are PSD; convexity in optimization.
- **LU decomposition** ($A = LU$): factor a square matrix into lower-triangular $L$ and upper-triangular $U$. **Why it matters**: solving $Ax=b$ becomes two triangular solves ($Ly=b$, then $Ux=y$) — each $O(n^2)$ once you have the factors. **In robotics**: solving the equations of motion $M(q)\ddot{q} = \tau - C(q,\dot{q})\dot{q} - G(q)$ for $\ddot{q}$ at every timestep; $M(q)$ is SPD so Cholesky ($LL^T$, the SPD special case of LU) is the fast path. Also appears in any least-squares normal-equation solve and in MPC QP solvers. **Partial pivoting** (PA=LU) is the numerically stable form — swaps rows to put the largest pivot on the diagonal.
  - LU vs Cholesky: Cholesky only works on SPD matrices but is ~2× faster and more numerically stable — always prefer it for inertia matrices.
- **Quadratic forms** $x^TMx$: the shape of every cost in [[LQR]]/[[MPC & Virtual Fixtures|MPC]] and the energy in dynamics.
- **Orthogonality / rotation matrices**: $R^TR=I$ ([[Frames & Rotations]]) — distance-preserving, inverse = transpose.
- **Rank / nullspace**: controllability/observability rank tests ([[State-Space & Pole Placement]]); redundant-arm IK uses the Jacobian nullspace for secondary objectives.


## Links

- Related: [[State-Space & Pole Placement]], [[Jacobian]], [[Inverse Kinematics & DLS]], [[LQR]], [[Kalman Filter]], [[Frames & Rotations]], [[Probability & Optimization]]
- Parent: [[00 Knowledge Map]]

---

