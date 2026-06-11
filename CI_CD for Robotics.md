---
type: concept
domain: SWE & math foundations
status: drafted
last-reviewed: 2026-06-02
tags: [ci-cd, github-actions, docker, testing, devops]
---

# CI/CD for Robotics


---

## Core idea

CI (Continuous Integration) builds and tests every commit automatically. CD (Continuous Delivery) packages or deploys the result. For robotics: CI catches broken builds and regressions before they reach the robot. The main challenge is that ROS2 packages need a full ROS environment — Docker solves this.

---

## GitHub Actions for ROS2

```yaml
# .github/workflows/build.yml
name: ROS2 Build & Test

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-22.04
    container:
      image: ros:humble      # official ROS2 Docker image

    steps:
      - uses: actions/checkout@v3

      - name: Install dependencies
        run: |
          apt-get update
          rosdep update
          rosdep install --from-paths src --ignore-src -r -y

      - name: Build
        run: |
          source /opt/ros/humble/setup.bash
          colcon build --packages-select wall_e_bringup

      - name: Test
        run: |
          source /opt/ros/humble/setup.bash
          source install/setup.bash
          colcon test --packages-select wall_e_bringup
          colcon test-result --verbose
```

---

## Why Docker for CI

- GitHub Actions runners don't have ROS2 installed
- `ros:humble` image gives a clean, reproducible ROS environment
- No dependency drift between dev machine and CI
- Same image used in CI can be deployed to the robot

---

## ROS2-specific CI steps

| Step | What it does |
|------|-------------|
| `rosdep install` | Installs all package.xml dependencies |
| `colcon build` | Compiles the package, catches CMake/C++ errors |
| `colcon test` | Runs unit tests (gtest) and launch tests |
| `colcon test-result` | Summarizes pass/fail |

---

## What to test in a ROS2 CI pipeline

```
Unit tests      → gtest for C++ logic (e.g. drive math, packet parsing)
Launch tests    → spin up nodes, check topics publish, check TF tree
Linting         → ament_clang_format, ament_flake8
URDF validation → check_urdf, xacro parsing
```

---

## Practical setup for WALL-E

```yaml
# Minimal useful CI for WALL-E hardware package
- Build wall_e_bringup
- Validate URDF: check_urdf + xacro
- Lint C++ nodes with ament_clang_format
- Run any unit tests for drive math / packet parsing
# NOT: hardware tests (no Arduino/Jetson in CI)
```

Hardware-in-the-loop tests run separately on the actual robot. CI only catches software regressions.

---

## Common pitfalls

| Pitfall | Fix |
|---------|-----|
| Forgetting `source /opt/ros/.../setup.bash` | Add to every step that uses ros2 commands |
| CI passes but robot fails | CI uses sim/mock; add hardware integration tests separately |
| Slow CI (full rebuild every push) | Use `actions/cache` on `build/` and `install/` directories |
| rosdep fails on private deps | Add custom rosdep sources or install manually |

---

## Links
- Related: [[Git & Build Systems]], [[Docker for ROS]], [[ROS2 Node Design Patterns]]
- Parent: [[00 Knowledge Map]]

---

