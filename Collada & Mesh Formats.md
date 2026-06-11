---
type: concept
domain: Simulation & tooling
status: drafted
last-reviewed: 2026-06-02
tags: [collada, urdf, mesh, cad, gazebo, rviz]
---

# Collada & Mesh Formats


---

## Core idea

Collada (`.dae`) is a 3D model interchange format used to represent complex robot geometry in URDF. Primitives (box, cylinder, sphere) are fast to write but look nothing like the real robot. When you need accurate appearance or collision geometry — export from CAD as `.dae`, reference it in the URDF.

---

## Primitives vs mesh — when to use each

| | Primitives | Mesh (Collada/STL) |
|--|-----------|-------------------|
| **Use when** | Prototyping, simple shapes, simulation-only | Real hardware shape, demos, accurate collision |
| **Complexity** | One line in URDF | Requires CAD export pipeline |
| **Simulation speed** | Fast (simple collision math) | Slower (complex geometry) |
| **Looks like real robot** | No | Yes |
| **WALL-E now** | ✓ (boxes + cylinders) | Future: SolidWorks export |

---

## URDF mesh reference

```xml
<link name="base_link">
  <visual>
    <geometry>
      <!-- Collada — full color, textures, complex shape -->
      <mesh filename="package://wall_e_bringup/meshes/wall_e_body.dae"/>
    </geometry>
  </visual>
  <collision>
    <!-- STL for collision — simpler, faster physics -->
    <geometry>
      <mesh filename="package://wall_e_bringup/meshes/wall_e_body_collision.stl"/>
    </geometry>
  </collision>
</link>
```

---

## Mesh formats comparison

| Format | Extension | Use |
|--------|-----------|-----|
| **Collada** | `.dae` | Visual mesh — stores color, texture, materials |
| **STL** | `.stl` | Collision mesh — geometry only, no color, lightweight |
| **OBJ** | `.obj` | Alternative visual mesh, widely supported |
| **glTF** | `.glb/.gltf` | Modern alternative to Collada, smaller files |

**Rule:** visual mesh → `.dae` (looks good). Collision mesh → `.stl` (fast physics). Never use a high-poly visual mesh for collision — it will tank simulation performance.

---

## CAD → URDF workflow

```
SolidWorks / Fusion 360
    ↓
Export assembly as URDF (sw_urdf_exporter / fusion2urdf plugin)
    ↓ generates:
  robot.urdf          ← joint/link structure
  meshes/part.dae     ← visual geometry per link
  meshes/part.stl     ← collision geometry per link
    ↓
Load into Gazebo / RViz / MoveIt
```

**sw_urdf_exporter** (SolidWorks): exports each part as a link, joints from mates
**fusion2urdf** (Fusion 360): similar, also generates ROS2-compatible URDF

---

## Collada file structure (what's inside)

```xml
<COLLADA>
  <asset>          <!-- units, author, created -->
  <library_geometries>   <!-- vertex data, triangles -->
  <library_materials>    <!-- material definitions -->
  <library_images>       <!-- texture references -->
  <library_visual_scenes> <!-- scene graph, transforms -->
</COLLADA>
```

Gazebo and RViz parse this to render the mesh. The important part is `library_geometries` (the actual 3D shape).

---

## Common issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Mesh not showing in RViz | Wrong package path | Use `package://pkg_name/meshes/file.dae` not absolute path |
| Robot looks correct in RViz but not Gazebo | Material not supported | Re-export or convert to simpler material |
| Simulation very slow | High-poly collision mesh | Create simplified collision STL (decimate in Blender) |
| Wrong scale | CAD units in mm, URDF in m | Scale by 0.001 in `<mesh>` tag: `<mesh filename="..." scale="0.001 0.001 0.001"/>` |
| Colors missing | Collada missing material | Add `<material>` tag in URDF or re-export with materials |

---

## Mesh placement in ROS2 package

```
my_robot_pkg/
├── meshes/
│   ├── body_visual.dae      ← Collada, full detail
│   ├── body_collision.stl   ← STL, simplified
│   ├── wheel_visual.dae
│   └── wheel_collision.stl
├── urdf/
│   └── robot.urdf.xacro
```

Add to `CMakeLists.txt`:
```cmake
install(DIRECTORY meshes urdf
  DESTINATION share/${PROJECT_NAME})
```

---

## Links
- Related: [[URDF & CAD Pipeline]], [[MuJoCo & Gazebo]], [[WALL-E V3]], [[FENCE-BOT]]
- Parent: [[00 Knowledge Map]]

---

