---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: tags: [ros2, colcon, ament, launch, build]

# colcon ament & launch

> [!question] Explain it cold
> 
> - What does colcon do vs ament vs CMake?
> - What's in a `package.xml` and why does build order matter?
> - How does a launch file differ from just running nodes by hand?

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


## Interview follow-ups

- **Q:** colcon vs ament vs CMake — who does what?
    - **A:** CMake is the underlying build generator; ament is ROS2's CMake/Python conventions and macros on top; colcon is the workspace-level tool that reads package.xml, computes dependency order, and invokes the per-package builds.
- **Q:** You built successfully but the node runs old behavior. Why?
    - **A:** Stale binaries / didn't re-source `install/setup.bash`, or built the wrong package. With `--symlink-install`, Python edits take effect without rebuild; C++ still needs a rebuild + re-source.
- **Q:** Why a launch file over a bash script of `ros2 run`s?
    - **A:** Declarative composition, parameter files, remappings, namespaces, ordered/conditional bring-up, and composable-node containers — all first-class, reproducible, and parameterizable.


## Links

- Related: [[ROS2 Node & Lifecycle]], [[Docker for ROS]], [[Git & Build Systems]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

colcon vs ament vs CMake — one line each? ? CMake = underlying build generator; ament = ROS2's CMake/Python conventions and macros; colcon = workspace tool that reads package.xml, orders dependencies, and builds all packages.

You rebuilt but the node still runs old behavior — likely cause? ? Stale binaries — didn't re-source install/setup.bash, or built the wrong package. (C++ needs rebuild + re-source; --symlink-install covers Python edits.)

What does package.xml drive in the build? ? Dependency declarations (build/exec/depend) that colcon uses to compute topological build order across the workspace.

Why use a launch file instead of multiple ros2 run commands? ? Declarative, reproducible bring-up: parameters, remappings, namespaces, composable-node containers, and ordered/conditional startup.