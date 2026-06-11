---

## type: project domain: Simulation & Tooling status: drafted last-reviewed: tags: [isaac-lab, ros2, vr-teleop, ik, pinn, cpp, python]

# FENCE-BOT

> [!question] Explain it cold
>
> - What does FENCE-BOT do end-to-end?
> - How does data flow from the VR headset to the robot arm in Isaac Lab?
> - What was the IK approach and why DLS?

---

## What it is

FENCE-BOT is a VR-teleoperated 6-DOF robotic arm simulation. A VR headset streams end-effector pose targets in real-time; a ROS2 middleware layer bridges the VR UDP stream into Isaac Lab, where a DLS inverse kinematics controller drives the ASEM V2 arm to track the target. A PINN-based contact estimator predicts contact forces from joint state before the sensor fires.

---

## Full data flow

```
VR Headset
  │  UDP (28 bytes = 7 floats: x,y,z,qx,qy,qz,qw)
  ▼  port 5005
vr_udp_publisher (ROS2 C++ node)
  │  /vr_pose  [geometry_msgs/PoseStamped]  best_effort QoS
  ▼
robot_controller (ROS2 C++ node, 50Hz timer)
  │  UDP (28 bytes, same struct)
  ▼  port 5006
run_sim.py (Isaac Lab)
  │  DifferentialIKController (DLS, λ=0.1)
  ▼
ASEM V2 6-DOF arm in Isaac Sim (PhysX)
  │  ContactSensor on link6 (EE)
  ▼
CSVLogger → logs/fencebot_<timestamp>.csv
```

**Packet struct (both UDP hops):**
```cpp
struct Packet { float x, y, z, qx, qy, qz, qw; };  // 28 bytes
```

---

## Node breakdown

### `vr_udp_publisher.cpp`
- Binds UDP socket on `0.0.0.0:5005`
- Background thread (`std::thread`) receives packets, converts to `PoseStamped`
- Publishes on `/vr_pose` with `best_effort` QoS (matches VR latency tolerance)
- Uses `std::atomic<bool> running_` for clean thread shutdown

### `robot_controller.cpp`
- Subscribes to `/vr_pose` (`best_effort` QoS — must match publisher)
- 50Hz `create_wall_timer` decouples control rate from VR publish rate
- `std::mutex` guards shared `latest_pose_` between callback and timer
- `new_data_` flag prevents re-sending stale poses every tick
- Opens raw UDP socket to Isaac Lab on `127.0.0.1:5006`
- `RCLCPP_INFO_THROTTLE` at 1000ms avoids 50 log lines/sec

### `run_sim.py` (Isaac Lab)
- `UDPListener` thread receives from port 5006, stores latest pose behind a `threading.Lock`
- `DifferentialIKController` config:
  ```python
  DifferentialIKControllerCfg(
      command_type="pose",
      use_relative_mode=False,
      ik_method="dls",
      ik_params={"lambda_val": 0.1}
  )
  ```
- IK computation:
  ```python
  jacobian = env.robot.root_physx_view.get_jacobians()[:, ee_idx - 1, :6, :6]
  actions = ik_controller.compute(ee_pos_curr, ee_quat_curr, jacobian, joint_pos)
  actions = torch.clamp(actions, -3.14, 3.14)
  ```
- Red cube target (`RigidObjectCfg`) floats at EE + 2cm offset in X — visual tracking reference
- Hit counter tracks rising-edge contact events (only counts first frame of contact)
- CSV logger records every sim step: target pose, actual EE, 6 joint positions, contact force vector + magnitude

---

## Isaac Lab environment (`vr_arm_env.py`)

```python
ASEM_CFG = ArticulationCfg(
    prim_path="/World/envs/env_.*/Robot",
    spawn=sim_utils.UsdFileCfg(usd_path=ASEM_USD_PATH, activate_contact_sensors=True),
    init_state=ArticulationCfg.InitialStateCfg(
        joint_pos={"j1": 0.0, "j2": 0.0, "j3": 0.0, "j4": 0.0, "j5": 0.0, "j6": 0.0}
    ),
    actuators={"all_joints": ImplicitActuatorCfg(
        joint_names_expr=["j1", "j2", "j3", "j4", "j5", "j6"],
        stiffness=800.0, damping=40.0
    )}
)
```

- `ImplicitActuatorCfg` — PhysX handles the PD control internally (stiffness=800, damping=40)
- `DirectRLEnv` base class — simpler than `ManagerBasedRLEnv`, direct tensor operations
- `ContactSensor` on `link6` (EE), 3-step history, 0.5N threshold for "hit"
- `ASEM_USD_PATH` resolves from env var or repo-relative path — fixed the hardcoded `/home/wukong/` bug

---

## Math utilities

### `DH_calculator.py`
Symbolic DH transform using SymPy. Implements Denavit-Hartenberg convention:
```
T = Rz(θ) · Tz(d) · Tx(a) · Rx(α)
```
```python
def get_dh_matrix(theta, d, a, alpha):
    T = Rz * Tz * Tx * Rx   # sympy matrix multiply
    return sp.simplify(T)
```
Used to validate the ASEM arm's FK against the Isaac Sim output.

### `urdf2dhparam.py`
Parses URDF joint definitions and extracts (θ, d, a, α) for each joint — automates the DH table from the robot description.

---

## Neural networks

### PINN Contact Predictor (`Pinn_Contact.py`)
Predicts contact force from joint state **before the sensor fires** — useful for anticipatory control.

**Architecture:**
```
Input (9): [ee_x, ee_y, ee_z, j1, j2, j3, j4, j5, j6]
    ↓
Backbone: Linear(9→64) → Tanh → Linear(64→64) → Tanh → Linear(64→32) → Tanh
    ├── Force vector head: Linear(32→3)           → [fx, fy, fz]
    └── Force magnitude head: Linear(32→1) → Softplus  → force_mag (always ≥ 0)
```

**Physics constraints in loss:**
1. Force magnitude non-negative (enforced by Softplus activation)
2. Force continuous w.r.t. EE position (gradient penalty in loss)
3. Free-space predictions near zero (zero-force loss on non-contact samples)

**Why Tanh over ReLU:** Smoother gradients help physics-constrained losses converge. ReLU's dead neuron problem is worse when gradient flow through the physics terms is critical.

### LNN (`train_lnn_fixed.py`)
Lagrangian Neural Network — structure-preserving ML that learns robot dynamics while respecting energy conservation (Hamiltonian mechanics). Used for contact estimation and dynamics prediction.

---

## Key design decisions (interview answers)

**Why UDP instead of native Isaac ROS2 bridge?**
> The native bridge adds latency through DDS serialization. UDP with a raw 28-byte struct is synchronous and low-latency — the VR teleop loop needs sub-20ms end-to-end. We accept UDP's unreliable delivery because dropping a pose frame just means the arm holds its current target for one tick.

**Why DLS (Damped Least Squares) IK?**
> Pure pseudoinverse IK diverges at singularities — the damping term λ=0.1 trades off exactness near singularities for stability. For teleoperation, stability matters more than exact target tracking at extreme poses.

**Why `best_effort` QoS on `/vr_pose`?**
> Reliable QoS would queue missed messages and deliver them late — worse for real-time teleop than just dropping them. Best_effort drops missed packets cleanly. Both publisher and subscriber must use the same QoS or the subscription silently receives nothing.

---

## Gotchas

- **Jacobian index:** `get_jacobians()[:, ee_idx - 1, ...]` — the jacobian index is body index minus 1 (off-by-one from Isaac's body indexing). This was a debugging pain point.
- **QoS mismatch:** Originally had reliable publisher / best_effort subscriber — subscription received nothing, no error thrown
- **USD path hardcoded:** Early version had `/home/wukong/...` hardcoded — broke on any other machine. Fixed with `FENCEBOT_USD_PATH` env var + repo-relative default
- **Contact rising-edge:** Without the `last_in_contact` flag, the hit counter increments every physics step during contact — one touch became 50+ "hits"
- **IK joint clamping:** Without `torch.clamp(actions, -3.14, 3.14)`, IK can produce joint targets outside physical range → arm teleports

---

## Links

- Related: [[Isaac Lab]], [[Inverse Kinematics & DLS]], [[VR Teleop Pipeline]], [[Contact Modeling]], [[ROS2 Node Design Patterns]], [[PINN Contact Estimation]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is the full data path in FENCE-BOT from VR to robot? ? VR → UDP:5005 (28 bytes) → vr_udp_publisher (ROS2) → /vr_pose → robot_controller (50Hz timer) → UDP:5006 → Isaac Lab run_sim.py → DLS IK → ASEM V2 joint targets → PhysX sim.

Why DLS IK with λ=0.1 instead of pseudoinverse? ? Pure pseudoinverse diverges at singularities (J becomes near-singular, joint velocities explode). DLS damping term λ=0.1 trades off exact tracking for stability near singularities — critical for teleop where you can't have the arm flail.

What's the Jacobian indexing gotcha in Isaac Lab? ? get_jacobians() returns shape (num_envs, num_bodies, 6, num_joints). The EE jacobian is at index ee_idx - 1 not ee_idx — off-by-one from Isaac's body indexing convention.

Why use Softplus activation for the force magnitude output in PINN? ? Force magnitude must always be ≥ 0 (physics constraint). Softplus(x) = log(1+eˣ) is a smooth, always-positive approximation of ReLU. Unlike ReLU it has smooth gradients everywhere — important when the physics loss terms need to backpropagate through this output.
