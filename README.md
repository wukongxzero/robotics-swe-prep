# Robotics SWE Interview Prep — Knowledge Vault

A comprehensive Obsidian vault for engineers targeting **Robotics Software Engineer (RSE)** roles at companies like Boston Dynamics, Waymo, Figure, Cruise, and similar.

97 pages covering the full technical stack — from DSA patterns to ROS2 internals to real-time Linux to RL simulation.

---

## What's inside

### DSA & Coding Screens
- **DSA Patterns** — NeetCode 150 in C++, pattern index, templates, flashcards (Two Pointers, Graphs, BFS/DFS, Sliding Window, Heap, Dijkstra, A*, Binary Search, DP)
- **Interview Problem-Solving Framework** — spoken approach, 5-step method

### ROS2
- Node design patterns, executors, lifecycle, QoS, comm patterns
- colcon/ament build system, rosbag2, ros2_control, TF2, Nav2, DDS & RMW
- Multi-machine ROS2, ROS2 testing

### Robotics Core
- Kalman Filter (Linear, EKF, UKF), Sensor Fusion, Complementary Filter
- Forward/Inverse Kinematics, Jacobian, DLS IK
- Motion Planning (A*, RRT, Dijkstra), SLAM & RTAB-Map
- PID, LQR, LQG, MPC, State-Space, Trajectory Optimization
- Robot Dynamics, Floating-Base & Whole-Body Control

### Systems & Infrastructure
- Linux IPC, real-time scheduling, concurrency, debugging tools (htop/btop, perf, strace, gdb)
- Docker for ROS, Jetson Orin setup, CUDA & CuPy, Edge Inference
- Git & Build Systems, snap, CI/CD for Robotics
- Safety-Critical Architecture, Real-Time Determinism, Latency Budgets

### Hardware & Embedded
- CAN Bus, Serial Packet Protocols, AVR, FreeRTOS, RTOS Fundamentals
- Motor Control, Dynamixel, Watchdog Timers, Parallax Propeller

### Simulation & RL
- Isaac Lab/Sim setup, ArticulationCfg, DirectRLEnv, ContactSensor, DLS IK
- MuJoCo & Gazebo, Simulation Environments Comparison

### Perception
- YOLOv8, OpenCV Pipelines, Point Cloud Processing, Camera Model & Projection
- URDF & CAD Pipeline, Collada & Mesh Formats

### Projects (real implementations)
- **WALL-E V3** — autonomous mobile robot, ROS2 + Nav2 + LLaMA + YOLO on Jetson Orin
- **FENCE-BOT** — 6-DOF arm in Isaac Lab with VR teleop, DLS IK, contact sensing

---

## How to use

1. Install [Obsidian](https://obsidian.md)
2. Clone this repo: `git clone https://github.com/wukongxzero/robotics-swe-prep`
3. Open Obsidian → Open folder as vault → select the cloned directory
4. Enable the **Spaced Repetition** plugin for flashcard drilling

Each page follows the same structure:
- **Explain it cold** callout — test yourself before reading
- Core concepts with C++ code examples
- Interview follow-ups with model answers
- Gotchas section
- `#flashcards` at the bottom for spaced repetition

---

## Who this is for

Engineers targeting RSE/robotics SWE roles who need to prep across:
- Coding screens (graphs, BFS/DFS, two pointers — the RSE-relevant patterns)
- Systems design (real-time, IPC, ROS2 architecture)
- Robotics fundamentals (kinematics, control, planning, estimation)

---

## Contributing

PRs welcome — especially for pages marked `status: stub` or gaps in coverage.

---

*Built with Obsidian. Maintained alongside active robotics projects.*
