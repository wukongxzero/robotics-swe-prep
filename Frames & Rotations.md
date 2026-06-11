---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [kinematics, rotations, se3, quaternions, transforms]

# Frames & Rotations

> [!question] Explain it cold
> 
> - What's in a homogeneous transform, and why 4×4?
> - Three ways to represent rotation — tradeoffs of each.
> - What is gimbal lock and which representation avoids it?

---

## Core idea

Robotics is bookkeeping of coordinate frames. A rigid-body pose = position + orientation, packed into a **homogeneous transformation matrix** $T \in SE(3)$ that lets you compose frames by multiplication. Get the frame algebra right and FK/IK/Jacobians fall out; get it wrong and every downstream sign error traces back here.

## Key facts & formulas

- **Rotation matrix** $R \in SO(3)$: 3×3 orthonormal, $R^TR=I$, $\det R = +1$. Columns are the rotated axes. Composes by multiplication; inverse = transpose.
- **Homogeneous transform** $T \in SE(3)$, 4×4: $$T = \begin{bmatrix} R & p \ 0\ 0\ 0 & 1 \end{bmatrix}$$ The 4×4 (not 3×4) so transforms _compose by matrix multiply_ and you can invert cleanly. $T_{0}^{2} = T_{0}^{1}T_{1}^{2}$ — chain frames by multiplying.
- **Inverse**: $T^{-1} = \begin{bmatrix} R^T & -R^Tp \ 0 & 1\end{bmatrix}$ (not a generic matrix inverse — exploit structure).
- **Rotation representations**:
    - **Euler angles** (roll/pitch/yaw): 3 numbers, intuitive, but suffer **gimbal lock** (lose a DOF when two axes align) and are order-dependent.
    - **Rotation matrix**: 9 numbers, no singularities, but redundant (6 constraints).
    - **Quaternion** (4 numbers, unit norm): no gimbal lock, compact, numerically stable, best for interpolation (SLERP) and integration. The default for orientation state. Double-cover: $q$ and $-q$ are the same rotation.
    - **Axis-angle / rotation vector** (3 numbers): minimal, great for representing small rotations / angular velocity integration.


## Interview follow-ups

- **Q:** Why a 4×4 homogeneous transform instead of storing R and p separately?
    - **A:** So poses compose by a single matrix multiply ($T_0^2 = T_0^1 T_1^2$) and chains of frames are clean to build and invert. It unifies rotation and translation into one operator.
- **Q:** What's gimbal lock?
    - **A:** With euler angles, when two rotation axes align you lose a rotational DOF — the representation degenerates. Quaternions avoid it because they don't decompose rotation into sequential axis rotations.
- **Q:** When quaternion vs rotation matrix?
    - **A:** Quaternion for orientation _state_, integration, and interpolation (compact, stable, no singularity). Rotation matrix when I need to actually transform vectors or compose with translations in an $SE(3)$ chain.


## Links

- Related: [[Forward Kinematics]], [[Jacobian]], [[Complementary Filter]], [[Serial Packet Protocols]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What's the structure of a homogeneous transform and why 4×4? ? T = [[R, p],[0,1]] with R∈SO(3), p the position. 4×4 so poses compose by matrix multiply (T₀² = T₀¹T₁²) and invert cleanly.

What is gimbal lock and which representation avoids it? ? With euler angles, when two axes align you lose a rotational DOF. Quaternions avoid it — they don't decompose rotation into sequential axis rotations.

Inverse of a homogeneous transform? ? T⁻¹ = [[Rᵀ, −Rᵀp],[0,1]] — exploit the structure, don't do a generic 4×4 inverse.

When do you use a quaternion vs a rotation matrix? ? Quaternion for orientation state, integration, and SLERP interpolation (compact, stable, no singularity); rotation matrix to transform vectors / compose in an SE(3) chain.