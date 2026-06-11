---
type: concept
domain: Kinematics & dynamics
status: drafted
last-reviewed:
tags: [kinematics, FK, DH, transforms]
---

# Forward Kinematics

> [!question] Explain it cold
> *Answer from memory first.*
>
> - What does FK compute, and what are the inputs/outputs?
> - What are DH parameters and what does each one mean?
> - How do you compose transforms across multiple joints?

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

## Where I've used it
- **Today's C++ problem**: 2-DOF planar FK — computed $(x,y)$ from $(\theta_1,\theta_2)$ using cumulative angle formula.
- **FENCE-BOT**: DLS IK requires the [[Jacobian]], which is $\partial\,\text{FK}/\partial q$ — correct FK is the foundation.

## Interview follow-ups
- **Q:** What's the difference between FK and IK, and which is harder?
  - **A:** FK is direct and always has a unique answer — plug in angles, get pose. IK is the inverse — find angles for a desired pose — and can have zero, one, or infinitely many solutions. IK is much harder; no closed form for >3 DOF in general.
- **Q:** For a 6-DOF arm, how many DH transforms do you compose?
  - **A:** Six — $T^0_6 = T^0_1 T^1_2 \cdots T^5_6$.
- **Q:** Why are the angles cumulative in a planar arm?
  - **A:** Each joint rotates relative to the previous link's frame, so the second link's world angle is $\theta_1 + \theta_2$.

## Gotchas / what trips me up
- Two DH variants exist (classic vs modified/Craig) — they differ in frame assignment order. Know which one your library uses before trusting its output.
- Getting the $z$-axis assignment wrong cascades through every transform downstream.

## Links
- Related: [[Jacobian]], [[Inverse Kinematics & DLS]], [[Frames & Rotations]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does forward kinematics compute?
?
Given joint angles q, it computes the end-effector pose (position + orientation) in the world frame.

Write the FK formula for a 2-DOF planar arm.
?
x = l1·cos(θ1) + l2·cos(θ1+θ2), y = l1·sin(θ1) + l2·sin(θ1+θ2)

Why is the second link's angle (θ1+θ2) and not just θ2?
?
Joint angles are cumulative — each is relative to the previous link's frame, so the world angle of link 2 is the sum of both joint angles.

Name the four DH parameters.
?
a (link length), α (link twist), d (joint offset), θ (joint angle — variable for revolute joints).

What is a homogeneous transform matrix?
?
A 4×4 matrix [[R, p],[0, 1]] where R is 3×3 rotation and p is 3D translation. Chains by matrix multiplication.
