---

## type: concept domain: SWE & math foundations status: drafted last-reviewed: tags: [math, linear-algebra]

# Linear Algebra Refresher

> [!question] Explain it cold
> 
> - What do eigenvalues/eigenvectors mean physically in robotics?
> - What is the SVD and why does it show up everywhere?
> - What does the pseudoinverse solve?
> - What is A=LU and when do you reach for it over other solvers?

---

## Core idea

Linear algebra is the substrate under all of robotics — transforms, control, estimation, IK, optimization. The goal isn't to re-derive it, but to _recognize_ the same handful of objects (eigenvalues, SVD, pseudoinverse, positive-definiteness) wherever they recur, because they connect notes across the whole vault.

## Key facts & where each recurs

- **Eigenvalues/eigenvectors** ($Av = \lambda v$): directions a transform only scales. **In robotics**: eigenvalues of $(A-BK)$ are the closed-loop poles ([[State-Space & Pole Placement]]) — stability = all in the left-half plane. Eigenvalues of the inertia matrix = principal inertias.
- **SVD** ($A = U\Sigma V^T$): any matrix as rotation–scale–rotation; singular values measure "gain" per direction. **In robotics**: [[Jacobian]] singularities = a singular value → 0; manipulability uses singular values; numerical rank; PCA. The Swiss-army knife.
- **Pseudoinverse** ($A^+$): best least-squares solution to over/underdetermined $Ax=b$. **In robotics**: the basis of numerical [[Inverse Kinematics & DLS|IK]] (and DLS is the _damped_ pseudoinverse).
- **Positive-definiteness** ($x^TMx > 0$): "energy is always positive." **In robotics**: inertia matrix $M(q)$ is SPD ([[Robot Dynamics Formulations]]); [[LQR]] cost weights $Q\succeq0, R\succ0$; covariance matrices ([[Kalman Filter]]) are PSD; convexity in optimization.
- **Quadratic forms** $x^TMx$: the shape of every cost in [[LQR]]/[[MPC & Virtual Fixtures|MPC]] and the energy in dynamics.
- **Orthogonality / rotation matrices**: $R^TR=I$ ([[Frames & Rotations]]) — distance-preserving, inverse = transpose.
- **Rank / nullspace**: controllability/observability rank tests ([[State-Space & Pole Placement]]); redundant-arm IK uses the Jacobian nullspace for secondary objectives.

## Interview follow-ups

- **Q:** What do eigenvalues tell you about a control system?
    - **A:** The eigenvalues of the closed-loop matrix (A−BK) are the poles — negative real parts mean stable, magnitude sets speed, imaginary parts mean oscillation. Stability analysis _is_ eigenvalue analysis.
- **Q:** Why is the SVD so ubiquitous?
    - **A:** It decomposes any matrix into rotation–scale–rotation and exposes the singular values (per-direction gain). It tells you rank, conditioning, and — for a Jacobian — proximity to singularity (a singular value going to zero). It underlies pseudoinverse, PCA, and numerical robustness.
- **Q:** What does positive-definiteness guarantee?
    - **A:** A quadratic form is always positive — physically, energy is positive; mathematically, invertibility and convexity. It's why the inertia matrix is invertible, why LQR/Kalman cost and covariance matrices behave, and why those optimizations are well-posed.

- **LU decomposition** ($A = LU$): factor a square matrix into lower-triangular $L$ and upper-triangular $U$. **Why it matters**: solving $Ax=b$ becomes two triangular solves ($Ly=b$, then $Ux=y$) — each $O(n^2)$ once you have the factors. **In robotics**: solving the equations of motion $M(q)\ddot{q} = \tau - C(q,\dot{q})\dot{q} - G(q)$ for $\ddot{q}$ at every timestep; $M(q)$ is SPD so Cholesky ($LL^T$, the SPD special case of LU) is the fast path. Also appears in any least-squares normal-equation solve and in MPC QP solvers. **Partial pivoting** (PA=LU) is the numerically stable form — swaps rows to put the largest pivot on the diagonal.


## Links

- Related: [[State-Space & Pole Placement]], [[Jacobian]], [[Inverse Kinematics & DLS]], [[LQR]], [[Kalman Filter]], [[Frames & Rotations]], [[Probability & Optimization]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What do the eigenvalues of (A−BK) tell you? ? They're the closed-loop poles: negative real part = stable, magnitude = speed, imaginary part = oscillation. Stability analysis is eigenvalue analysis.

Why does the SVD show up everywhere in robotics? ? It decomposes any matrix into rotation–scale–rotation, exposing singular values (per-direction gain) — giving rank, conditioning, Jacobian singularity proximity, pseudoinverse, and PCA.

What does positive-definiteness (xᵀMx>0) guarantee, and where does it recur? ? Always-positive quadratic form → invertibility and convexity. Recurs in the SPD inertia matrix, LQR cost weights, Kalman covariances, and well-posed optimization.

What is A=LU and when do you use Cholesky instead? ? LU factors a square matrix into lower × upper triangular, turning Ax=b into two O(n²) triangular solves. Use it for general square systems. Use Cholesky (A=LLᵀ) when A is SPD (e.g. inertia matrix M(q)) — it's ~2× faster and numerically stabler. In robotics: solving M(q)q̈ = τ - C - G at every timestep uses Cholesky on M(q).