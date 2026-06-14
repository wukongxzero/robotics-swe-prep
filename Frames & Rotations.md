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

## Where I've used it

- **FENCE-BOT**: the whole `/vr_pose` → `robot_controller` → Isaac Lab pipeline is transform bookkeeping — VR controller pose in one frame, mapped to the ASEM arm's base/tool frames. Pose errors there are frame errors.
- **WALL-E gimbal**: attitude as euler (eulerX/Y/Z in the [[Serial Packet Protocols|TankStatus packet]]) — and euler is exactly where gimbal lock and the [[Complementary Filter]]'s angle handling matter.

## Interview follow-ups

- **Q:** Why a 4×4 homogeneous transform instead of storing R and p separately?
    - **A:** So poses compose by a single matrix multiply ($T_0^2 = T_0^1 T_1^2$) and chains of frames are clean to build and invert. It unifies rotation and translation into one operator.
- **Q:** What's gimbal lock?
    - **A:** With euler angles, when two rotation axes align you lose a rotational DOF — the representation degenerates. Quaternions avoid it because they don't decompose rotation into sequential axis rotations.
- **Q:** When quaternion vs rotation matrix?
    - **A:** Quaternion for orientation _state_, integration, and interpolation (compact, stable, no singularity). Rotation matrix when I need to actually transform vectors or compose with translations in an $SE(3)$(special euclidian group in 3x3) chain.

## Rotation matrices about principal axes (Rx, Ry, Rz)

The three elementary rotations. The identity matrix appears when θ = 0 — confirming no rotation.

**Rz(θ) — rotate about Z:**
```
Rz(θ) = [ cos θ  -sin θ  0 ]
         [ sin θ   cos θ  0 ]
         [ 0       0      1 ]
```

**Rx(θ) — rotate about X:**
```
Rx(θ) = [ 1    0       0   ]
         [ 0   cos θ  -sin θ]
         [ 0   sin θ   cos θ]
```

**Ry(θ) — rotate about Y:**
```
Ry(θ) = [ cos θ   0   sin θ]
         [  0      1    0   ]
         [-sin θ   0   cos θ]
```

**Memory trick for sign layout:** Think of rotating the unit vector `(1,0,0)` by θ around Z — it lands at `(cos θ, sin θ, 0)`. That's the **first column** of Rz. The negative sin is always top-right off-diagonal.

**Ry is the odd one out** — the sin signs are swapped compared to Rx/Rz because Y = Z×X (cyclic permutation reverses the cross product sign).

**Applying Rz to a 2D point like (4, 3):**
```python
import numpy as np
theta = np.pi / 4  # 45°
Rz = np.array([[np.cos(theta), -np.sin(theta), 0],
               [np.sin(theta),  np.cos(theta), 0],
               [0,              0,             1]])
p = np.array([4, 3, 0])
p_rot = Rz @ p  # rotates (4,3) in the XY plane
```

**Composing rotations:** multiply matrices — order matters (not commutative).
```python
R_total = Rz @ Ry @ Rx   # apply Rx first, then Ry, then Rz
```

---

## Gotchas / what trips me up

- Euler angle order/convention mismatches (XYZ vs ZYX) — silent, catastrophic.
- Forgetting the structured inverse and doing a full 4×4 inversion.
- Quaternion double-cover(when two rotations in quaternions are same) ($q \equiv -q$) biting interpolation/learning. 

## Links

- Related: [[Forward Kinematics]], [[Jacobian]], [[Complementary Filter]], [[Serial Packet Protocols]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What's the structure of a homogeneous transform and why 4×4? ? T = [[R, p],[0,1]] with R∈SO(3), p the position. 4×4 so poses compose by matrix multiply (T₀² = T₀¹T₁²) and invert cleanly.

What is gimbal lock and which representation avoids it? ? With euler angles, when two axes align you lose a rotational DOF. Quaternions avoid it — they don't decompose rotation into sequential axis rotations.

Inverse of a homogeneous transform? ? T⁻¹ = [[Rᵀ, −Rᵀp],[0,1]] — exploit the structure, don't do a generic 4×4 inverse.

When do you use a quaternion vs a rotation matrix? ? Quaternion for orientation state, integration, and SLERP interpolation (compact, stable, no singularity); rotation matrix to transform vectors / compose in an SE(3) chain.

Write Rz(θ) from memory. ? Rz = [[cosθ, -sinθ, 0],[sinθ, cosθ, 0],[0, 0, 1]]. Top-right off-diagonal is -sin. Memory: rotating (1,0,0) by θ gives (cosθ, sinθ, 0) — that's the first column.

Why is Ry's sin sign layout different from Rx and Rz? ? Because Y = Z×X — the cyclic permutation reverses the cross-product sign, flipping the -sin to the bottom-left instead of top-right.

How do you rotate a point (4, 3) in the XY plane by θ? ? Extend to 3D: p = [4, 3, 0], then p_rot = Rz(θ) @ p. The z-component stays 0.