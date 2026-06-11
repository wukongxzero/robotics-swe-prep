---
type: concept
domain: Kinematics & dynamics
status: drafted
last-reviewed:
tags: [kinematics, FK, DH, transforms]
---

# Forward Kinematics


---

## Core idea

Forward kinematics answers: **given the joint angles, where is the end-effector?** You chain together a sequence of rigid-body transforms — one per joint — from the base frame to the tool frame. The result is a 4×4 homogeneous transform $T^0_n$ giving the end-effector's position and orientation in the world.

## Key facts & formulas

### Homogeneous transform
$$T = \begin{bmatrix} R & p \\ 0 & 1 \end{bmatrix}$$
- $R \in SO(3)$: 3×3 rotation matrix
- $p \in \mathbb{R}^3$: translation vector
- Chaining: $T^0_n = T^0_1 \cdot T^1_2 \cdots T^{n-1}_n$ — multiply left to right along the chain.

### Planar 2-DOF arm (solved in C++ today)
With link lengths $l_1$, $l_2$ and joint angles $\theta_1$, $\theta_2$:
$$x = l_1\cos\theta_1 + l_2\cos(\theta_1+\theta_2)$$
$$y = l_1\sin\theta_1 + l_2\sin(\theta_1+\theta_2)$$
Key insight: $\theta_1 + \theta_2$ because joint angles are **cumulative** — each is measured relative to the previous link, not the world.

### Denavit-Hartenberg (DH) parameters
Four parameters per joint fully describe the transform between consecutive frames:

| Parameter | Meaning |
|-----------|---------|
| $a_i$ | link length (distance along $x_i$) |
| $\alpha_i$ | link twist (rotation about $x_i$) |
| $d_i$ | joint offset (distance along $z_{i-1}$) |
| $\theta_i$ | joint angle (rotation about $z_{i-1}$) — **variable for revolute joints** |

DH transform per joint:
$$T^{i-1}_i = R_z(\theta_i)\, T_z(d_i)\, T_x(a_i)\, R_x(\alpha_i)$$

- Revolute joint: $\theta_i$ varies, $d_i$ fixed
- Prismatic joint: $d_i$ varies, $\theta_i$ fixed


## Links
- Related: [[Jacobian]], [[Inverse Kinematics & DLS]], [[Frames & Rotations]]
- Parent: [[00 Knowledge Map]]

---

