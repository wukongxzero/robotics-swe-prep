---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [kinematics, rotations, se3, quaternions, transforms]

# Frames & Rotations


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


## Links

- Related: [[Forward Kinematics]], [[Jacobian]], [[Complementary Filter]], [[Serial Packet Protocols]]
- Parent: [[00 Knowledge Map]]

---

