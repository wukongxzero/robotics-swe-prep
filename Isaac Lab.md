---

## type: concept domain: Simulation & tooling status: drafted last-reviewed: tags: [sim, isaac, fence-bot, rl, cpp, python]

# Isaac Lab


---

## Core idea

Isaac Sim is NVIDIA's GPU-accelerated robotics simulator (built on Omniverse/USD with the PhysX engine); Isaac Lab is the RL/robot-learning framework layered on it. The headline feature is **massively parallel simulation** — thousands of robot instances on the GPU at once — plus high-fidelity rendering for sim-to-real perception.

## Key facts & formulas

- **Built on**: Omniverse + USD (Universal Scene Description) for the scene, **PhysX** for physics, GPU for both sim and rendering.
- **Parallel envs**: simulate thousands of environments simultaneously on-GPU — the reason it's used for RL/data generation (orders of magnitude more samples/sec than CPU sims).
- **USD scene**: assets, robots, and objects described in USD; spawn objects programmatically (e.g. `RigidObjectCfg` for a rigid body).
- **ROS2 bridge**: Isaac talks to a ROS2 stack over a bridge — in my case a **UDP** link, not the built-in ROS2 bridge, for the teleop loop.
- **vs MuJoCo/Gazebo**: Isaac wins on GPU parallelism + photorealistic rendering (perception sim-to-real); [[MuJoCo & Gazebo|MuJoCo]] wins on fast accurate contact for control research; Gazebo is the classic ROS-integrated workhorse.

---

## Setup (Ubuntu 24.04, RTX 5050 Blackwell)

```bash
# 1. Create conda env with Python 3.10 (Isaac Sim requires this)
conda create -n isaaclab python=3.10 -y
conda activate isaaclab

# 2. Accept conda ToS (required for non-interactive install)
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

# 3. Clone and install
git clone https://github.com/isaac-sim/IsaacLab.git ~/IsaacLab
cd ~/IsaacLab
OMNI_KIT_ACCEPT_EULA=YES ./isaaclab.sh --install
```

**Key env var:** `OMNI_KIT_ACCEPT_EULA=YES` — must be set or the kit installer calls `input()` in non-interactive mode and crashes with EOFError. Add to `.bashrc`.

**Driver only — no separate CUDA toolkit needed.** Isaac Sim ships its own CUDA runtime; host driver 595+ (Blackwell/RTX 5050) is sufficient.

---

## Known install pitfalls (2026-06-09)

**Python version:** Isaac Sim requires **Python 3.10**. Running `./isaaclab.sh --install` from a Python 3.13 base env will fail silently — `isaacsim` has no cp313 wheel on PyPI. Always create a fresh conda env first.

**RL dependency resolution too deep:** Running `./isaaclab.sh --install` (default = all) pulls in `rl-games → ray`, which causes `error: resolution-too-deep` from pip. Fix: use `./isaaclab.sh --install none` to skip RL frameworks and get the base simulator working first. Add RL frameworks separately later.

**Install order that works (verified 2026-06-09):**
```bash
conda create -n isaaclab python=3.10 -y
conda activate isaaclab
pip install 'isaacsim[all]==4.5.0' --extra-index-url https://pypi.nvidia.com  # ~20GB, Kit runtime
cd ~/IsaacLab && ./isaaclab.sh --install none   # IsaacLab extensions on top
```

Note: `isaacsim-rl` alone is NOT enough — it installs only the thin launcher, not `omni.kit.*`. You need `isaacsim[all]` for the full Kit runtime.

---

## ArticulationCfg — robot definition

```python
from isaaclab.assets import ArticulationCfg
import isaaclab.sim as sim_utils
from isaaclab.actuators import ImplicitActuatorCfg

ASEM_CFG = ArticulationCfg(
    prim_path="/World/envs/env_.*/Robot",
    spawn=sim_utils.UsdFileCfg(
        usd_path=ASEM_USD_PATH,          # from env var or repo-relative path
        activate_contact_sensors=True,   # required to enable ContactSensor
    ),
    init_state=ArticulationCfg.InitialStateCfg(
        joint_pos={"j1": 0.0, "j2": 0.0, "j3": 0.0,
                   "j4": 0.0, "j5": 0.0, "j6": 0.0}
    ),
    actuators={
        "all_joints": ImplicitActuatorCfg(
            joint_names_expr=["j1", "j2", "j3", "j4", "j5", "j6"],
            stiffness=800.0,   # position gain (PD built into PhysX)
            damping=40.0       # velocity damping
        )
    }
)
```

**`ImplicitActuatorCfg`**: PhysX handles PD control internally at the physics engine level. You send position targets; PhysX computes torques. Fast and stable for arm control.

---

## DirectRLEnv — environment base class

```python
from isaaclab.envs import DirectRLEnv, DirectRLEnvCfg

class VrArmEnv(DirectRLEnv):
    cfg: DirectRLEnvCfg

    def _setup_scene(self):
        self.robot = Articulation(self.cfg.robot_cfg)
        self.contact_sensor = ContactSensor(self.cfg.contact_sensor_cfg)
        self.scene.articulations["robot"] = self.robot
        self.scene.sensors["contact"] = self.contact_sensor

    def _pre_physics_step(self, actions: torch.Tensor):
        self.robot.set_joint_position_target(actions)

    def _get_observations(self):
        joint_pos = self.robot.data.joint_pos
        ee_pos = self.robot.data.body_pos_w[:, self.ee_idx]
        return {"policy": torch.cat([joint_pos, ee_pos], dim=-1)}

    def _get_rewards(self):
        ...
```

**`DirectRLEnv` vs `ManagerBasedRLEnv`**: DirectRLEnv is simpler — direct tensor ops, no manager overhead. Use it for custom research envs. ManagerBasedRLEnv has built-in observation/reward/action managers for standard RL pipelines.

---

## ContactSensor

```python
from isaaclab.sensors import ContactSensorCfg, ContactSensor

contact_sensor_cfg = ContactSensorCfg(
    prim_path="/World/envs/env_.*/Robot/link6",
    history_length=3,        # stores last N steps
    track_air_time=False,
    filter_prim_paths_expr=["/World/envs/env_.*/Cube"],  # what to detect contact with
)

# In the env loop:
contact_forces = self.contact_sensor.data.net_forces_w  # (num_envs, 3)
contact_mag = torch.norm(contact_forces, dim=-1)         # (num_envs,)
in_contact = contact_mag > 0.5  # threshold in Newtons
```

**Rising-edge detection** (counts one hit per contact event, not every frame):
```python
new_contact = in_contact & ~self.last_in_contact   # rising edge only
self.hit_counter += new_contact.float()
self.last_in_contact = in_contact
```

---

## DifferentialIKController (DLS)

```python
from isaaclab.controllers import DifferentialIKController, DifferentialIKControllerCfg

ik_cfg = DifferentialIKControllerCfg(
    command_type="pose",
    use_relative_mode=False,
    ik_method="dls",
    ik_params={"lambda_val": 0.1}   # DLS damping — stability near singularities
)
ik_controller = DifferentialIKController(ik_cfg, num_envs=1, device="cuda")

# Per step:
# Get Jacobian at EE body — NOTE: index is ee_body_idx - 1 (off-by-one!)
jacobian = robot.root_physx_view.get_jacobians()[:, ee_idx - 1, :6, :6]

# Compute joint targets
ee_pos, ee_quat = robot.data.body_pos_w[:, ee_idx], robot.data.body_quat_w[:, ee_idx]
joint_targets = ik_controller.compute(ee_pos, ee_quat, jacobian, robot.data.joint_pos)
joint_targets = torch.clamp(joint_targets, -3.14, 3.14)  # must clamp or arm teleports
```

**Why DLS?** Pure pseudoinverse diverges near singularities (J ill-conditioned → joint velocities explode). `lambda_val=0.1` trades off exact EE tracking for stability — essential for teleop.

---

## RigidObjectCfg (spawning objects)

```python
from isaaclab.assets import RigidObjectCfg
import isaaclab.sim as sim_utils

cube_cfg = RigidObjectCfg(
    prim_path="/World/envs/env_.*/Cube",
    spawn=sim_utils.CuboidCfg(
        size=(0.05, 0.05, 0.05),
        rigid_props=sim_utils.RigidBodyPropertiesCfg(),
        mass_props=sim_utils.MassPropertiesCfg(mass=0.1),
        visual_material=sim_utils.PreviewSurfaceCfg(diffuse_color=(1.0, 0.0, 0.0)),
    ),
    init_state=RigidObjectCfg.InitialStateCfg(pos=(0.5, 0.0, 0.5)),
)
```

---


---


## Links

- Related: [[FENCE-BOT]], [[VR Teleop Pipeline]], [[Inverse Kinematics & DLS]], [[Contact Modeling]], [[MuJoCo & Gazebo]], [[MPC & Virtual Fixtures]], [[Simulation Environments Comparison]]
- Parent: [[00 Knowledge Map]]

---

