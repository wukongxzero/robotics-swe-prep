# Robotics SWE Knowledge Vault

A reference vault for Robotics Software Engineers. Covers the full technical stack — DSA patterns in C++, ROS2 internals, real-time Linux, control theory, motion planning, and simulation.

Built for engineers working in or targeting robotics SWE roles.

---

## What's inside

### DSA & Algorithms
- **DSA Patterns** — C++ templates for Two Pointers, Graphs, BFS/DFS, Sliding Window, Heap, Dijkstra, A*, Binary Search, DP
- Graph algorithms with robotics context (path planning, dependency resolution)

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
- Git & Build Systems, CI/CD for Robotics
- Safety-Critical Architecture, Real-Time Determinism, Latency Budgets

### Hardware & Embedded
- CAN Bus, Serial Packet Protocols, AVR, FreeRTOS, RTOS Fundamentals
- Motor Control, Dynamixel, Watchdog Timers

### Simulation & RL
- Isaac Lab/Sim — ArticulationCfg, DirectRLEnv, ContactSensor, DLS IK
- MuJoCo & Gazebo, Simulation Environments Comparison

### Perception
- YOLOv8, OpenCV Pipelines, Point Cloud Processing, Camera Model & Projection
- URDF & CAD Pipeline, Collada & Mesh Formats

---

## How to use

1. Install [Obsidian](https://obsidian.md)
2. Clone this repo: `git clone https://github.com/wukongxzero/robotics-swe-prep`
3. Open Obsidian → Open folder as vault → select the cloned directory

Each page covers: core concepts, code examples, key formulas, and common pitfalls.

---

## Contributing

PRs welcome — especially for gaps in coverage or pages marked `status: stub`.
