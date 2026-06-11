---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: tags: [ros2, colcon, ament, launch, build]

# colcon ament & launch


---

## Core idea

The build-and-bring-up toolchain: **ament** is the ROS2 build system (CMake/Python macros + package conventions), **colcon** is the meta-build tool that resolves dependency order and builds many packages in one workspace, and **launch** is the orchestration system that starts/configures/wires nodes declaratively.

## Key facts & formulas

- **ament_cmake** — CMake extensions for ROS2 C++ packages: `ament_target_dependencies()`, `ament_export_*`, install rules. `ament_python` for Python packages.
- **colcon build** — builds the whole workspace, computes dependency graph from `package.xml`, builds in topological order, supports `--packages-select`, `--symlink-install` (edit Python without rebuilding).
- **package.xml** — declares `<depend>`, `<build_depend>`, `<exec_depend>`; this is what colcon reads to order the build. Wrong/missing deps → build-order bugs.
- **Workspace overlay**: `source install/setup.bash` layers your workspace over the underlying ROS install. Forgetting to re-source after a build = running stale binaries (I've hit this).
- **Launch** — Python (or XML/YAML) files that declare nodes, parameters, remappings, namespaces, composition containers, and conditionals. Lets you bring up a whole system with one command and parameterize it.
- **Launch features**: `Node(...)`, parameter files (YAML), `remappings`, `LaunchConfiguration` + `DeclareLaunchArgument` for args, `ComposableNodeContainer` for [[ROS2 Node & Lifecycle]] composition, event handlers for ordered bring-up.


## Links

- Related: [[ROS2 Node & Lifecycle]], [[Docker for ROS]], [[Git & Build Systems]]
- Parent: [[00 Knowledge Map]]

---

