---

## type: concept domain: Kinematics & dynamics status: drafted last-reviewed: tags: [dynamics, lagrangian, newton-euler, hamiltonian, kim]

# Robot Dynamics Formulations


---

## Core idea

Dynamics relates **forces/torques ↔ motion** (kinematics ignored forces; dynamics adds mass, inertia, gravity). Three equivalent formulations derive the same equations of motion from different starting points: energy (Lagrangian), force balance (Newton–Euler), or phase-space (Hamiltonian). Prof. Kim's Advanced Robotics covered all three plus floating-base.

## The manipulator equation (the one to know)

$$M(q)\ddot{q} + C(q,\dot{q})\dot{q} + g(q) = \tau$$

- $M(q)$ — **mass/inertia matrix** (configuration-dependent, symmetric positive-definite).
- $C(q,\dot{q})\dot{q}$ — **Coriolis and centrifugal** terms (velocity-dependent coupling).
- $g(q)$ — **gravity** torque.
- $\tau$ — applied joint torques (add $J^TF_{ext}$ for external/contact forces — links to [[Jacobian]] and [[Contact Modeling]]).

## The three formulations

- **Lagrangian** ($\frac{d}{dt}\frac{\partial L}{\partial \dot q} - \frac{\partial L}{\partial q} = \tau$, with $L = T - V$): energy-based (kinetic minus potential). Systematic, great for _deriving_ the symbolic equations of motion, scalar bookkeeping. What I used (via SymPy) for the LNN work. Best for analysis and getting clean $M, C, g$.
- **Newton–Euler**: force/torque balance on each link, done **recursively** (outward for velocities/accelerations, inward for forces). Computationally efficient — the standard for _real-time inverse dynamics_ (O(n)). Best for fast numerical computation.
- **Hamiltonian** ($H = T + V$, work in position+momentum phase space): energy-conserving structure, nice for analysis, optimal control, and energy-based/structure-preserving methods. Underlies some learned-dynamics and symplectic-integrator work.

## Forward vs inverse dynamics

- **Inverse dynamics**: given desired motion $(q,\dot q,\ddot q)$, find required $\tau$. Used for feedforward control / computed-torque. Newton–Euler does this in O(n).
- **Forward dynamics**: given $\tau$, find resulting $\ddot q$ (solve $M^{-1}(\tau - C\dot q - g)$). Used for **simulation** (then integrate). This is what physics engines like [[MuJoCo & Gazebo|MuJoCo]] / Isaac do.


## Links

- Related: [[Jacobian]], [[Lagrangian Neural Networks]], [[Floating-Base & Whole-Body]], [[Contact Modeling]], [[MPC & Virtual Fixtures]], [[Trajectory Optimization]]
- Parent: [[00 Knowledge Map]]

---

