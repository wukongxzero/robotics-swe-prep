---

## type: concept domain: Robot Modeling status: drafted last-reviewed: tags: [urdf, cad, fusion360, ros2, modeling]

# URDF & CAD Pipeline


---

## Core idea

URDF (Unified Robot Description Format) is an XML format that fully describes a robot: its **links** (rigid bodies with geometry, inertia, visual/collision meshes) and **joints** (how links connect and move). Every simulator and ROS2 tool consumes URDF — it's the universal robot model format.

---

## URDF structure

```xml
<robot name="my_robot">

  <!-- BASE LINK -->
  <link name="base_link">
    <visual>
      <geometry><mesh filename="package://my_robot/meshes/base.stl"/></geometry>
      <material name="gray"><color rgba="0.5 0.5 0.5 1"/></material>
    </visual>
    <collision>
      <geometry><mesh filename="package://my_robot/meshes/base_collision.stl"/></geometry>
    </collision>
    <inertial>
      <mass value="1.5"/>
      <origin xyz="0 0 0.05" rpy="0 0 0"/>
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.005"/>
    </inertial>
  </link>

  <!-- CHILD LINK -->
  <link name="link1"> ... </link>

  <!-- JOINT -->
  <joint name="joint1" type="revolute">
    <parent link="base_link"/>
    <child link="link1"/>
    <origin xyz="0 0 0.1" rpy="0 0 0"/>   <!-- joint frame relative to parent -->
    <axis xyz="0 0 1"/>                     <!-- rotation axis -->
    <limit lower="-1.57" upper="1.57" effort="10" velocity="1.0"/>
    <dynamics damping="0.1" friction="0.01"/>
  </joint>

</robot>
```

**Joint types:**
| Type | Description |
|------|-------------|
| `revolute` | Rotates about axis, has limits |
| `continuous` | Rotates about axis, no limits (wheels) |
| `prismatic` | Translates along axis, has limits |
| `fixed` | No motion (rigid attachment) |
| `floating` | 6-DOF free joint |
| `planar` | Motion in a plane |

---

## Key facts

- **Link tree**: URDF is a tree (no loops) rooted at `base_link`. Closed chains require mimic joints or SRDF workarounds.
- **Inertia matters**: Wrong inertia tensors → physics instability (objects spinning, exploding). Must be physically valid (positive definite).
- **Collision vs visual mesh**: Visual = high-poly for rendering. Collision = simplified convex hull for fast physics. Always separate them.
- **package:// paths**: Mesh paths use `package://pkg_name/meshes/file.stl` — requires the package to be in `$ROS_PACKAGE_PATH` / colcon workspace.
- **`xacro`**: URDF macro language — use it for any robot with repeated links (arms, legs). Avoids copy-paste URDF hell.

---

## Manual URDF workflow (what actually works)

```
1. Sketch the kinematic chain on paper — list all links and joints
2. Decide joint frames: each joint origin is relative to its parent link
3. Write link by link, test with:
      ros2 run urdf_tutorial display.py --urdf /path/to/robot.urdf
   or check_urdf robot.urdf
4. Add visual meshes (STL/DAE) — start with simple boxes/cylinders first
5. Add collision geometry — simplify meshes to convex hulls
6. Add inertial properties — compute from CAD or use formula for primitives
7. Add joint limits, dynamics (damping/friction)
8. Test in RViz2 with joint_state_publisher_gui
9. Load into simulator (Isaac/Gazebo/MuJoCo)
```

**Inertia for primitive shapes (useful for manual URDF):**
```
Solid box (m, x, y, z):   Ixx = m/12*(y²+z²),  Iyy = m/12*(x²+z²),  Izz = m/12*(x²+y²)
Solid cylinder (m, r, h): Ixx = Iyy = m/12*(3r²+h²),  Izz = m/2*r²
Solid sphere (m, r):      Ixx = Iyy = Izz = 2/5*m*r²
```

---

## fusion2urdf workflow

For Fusion 360 exports when CAD model exists:

```
1. Install fusion2urdf plugin in Fusion 360
2. Set up joint hierarchy in Fusion (as-built joints → component joints)
3. Assign rigid groups for links
4. Run fusion2urdf → generates URDF + STL meshes
5. Fix up output:
   - Check joint axes (often need manual correction)
   - Verify inertia values (plugin estimates, can be off)
   - Simplify collision meshes (exported as visual quality by default)
   - Fix package:// paths in mesh references
6. Test with check_urdf and RViz2
```

**fusion2urdf gotchas:**
- Joint axes come out rotated — always verify against your design intent
- Inertia tensors from the plugin are estimates; compute them in Fusion's mass properties for accuracy
- Mesh coordinate frames may be offset from joint origin — need `<origin>` adjustment in URDF
- Large assemblies → large STL files → slow simulation. Decimate meshes before using.

---

## Loading URDF into simulators

**ROS2 / RViz2:**
```bash
ros2 run robot_state_publisher robot_state_publisher --ros-args -p robot_description:="$(cat robot.urdf)"
ros2 run joint_state_publisher_gui joint_state_publisher_gui
ros2 run rviz2 rviz2
```

**Isaac Lab (ArticulationCfg):**
```python
from isaaclab.assets import ArticulationCfg
import isaaclab.sim as sim_utils

ROBOT_CFG = ArticulationCfg(
    spawn=sim_utils.UrdfFileCfg(
        asset_path="/path/to/robot.urdf",
        fix_base=True,               # fixed base arm
        make_instanceable=True,      # for parallel envs
    ),
    init_state=ArticulationCfg.InitialStateCfg(
        joint_pos={"joint1": 0.0, "joint2": 0.0},
    ),
)
```

**MuJoCo:**
```bash
python -c "import mujoco; mujoco.MjModel.from_xml_path('robot.urdf')"
# or convert to MJCF first:
python -m mujoco.mjcf.convert robot.urdf robot.xml
```

**Gazebo:**
```xml
<!-- in launch file -->
<node pkg="gazebo_ros" exec="spawn_entity.py"
      args="-topic robot_description -entity my_robot"/>
```

---

## Common failure modes

| Problem | Cause | Fix |
|---------|-------|-----|
| Robot explodes in sim | Bad inertia (non-positive-definite) | Recompute inertia; add small diagonal if zero |
| Mesh not found | Wrong `package://` path | Verify package name and ROS_PACKAGE_PATH |
| Joint moves wrong axis | Wrong `<axis xyz>` | Check axis against right-hand rule |
| Mesh rotated/offset | Frame mismatch between CAD origin and joint origin | Add `<origin>` to `<visual>` tag |
| Collision tunneling | Collision mesh too coarse or sim timestep too large | Decimate mesh; reduce dt in simulator |
| fusion2urdf joint wrong | Plugin misreads Fusion joint hierarchy | Re-parent joints in Fusion, re-export |

---

## Tools

| Tool | Purpose |
|------|---------|
| `check_urdf` | Validates URDF tree structure |
| `urdf_to_graphviz` | Visualizes kinematic chain as graph |
| `joint_state_publisher_gui` | Interactive joint slider testing in RViz2 |
| `xacro` | URDF macro preprocessor |
| `fusion2urdf` | Fusion 360 → URDF exporter |
| `MeshLab` / Blender | Mesh decimation and repair |
| `trimesh` (Python) | Programmatic mesh processing, inertia computation |

---

## Links

- Related: [[Simulation Environments Comparison]], [[Isaac Lab]], [[MuJoCo & Gazebo]], [[Forward Kinematics]], [[Inverse Kinematics & DLS]]
- Parent: [[00 Knowledge Map]]

---

