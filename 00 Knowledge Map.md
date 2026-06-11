---
type: moc
status: skeleton
created: 2026-05-24
---

# 00 — Robotics Knowledge Map

This is the front door to the vault. Every concept I know lives somewhere below. The job here is *capture* (so nothing decays) and *recall* (so I can say it out loud in an interview). If a note doesn't serve the portfolio or the drilling, it doesn't get written.

> [!tip] How to use this vault
> 1. Each leaf below becomes its own note using `[[_Note Template]]`.
> 2. Notes are recall-shaped: answer the "explain it cold" prompt from memory *before* reading the reference.
> 3. `#flashcards` blocks feed the Spaced Repetition plugin. Review beats re-reading.
> 4. `status::` frontmatter goes `skeleton → drafted → drilled`. Only `drilled` counts.

---

## Mermaid overview

```mermaid
mindmap
  root((Robotics))
    ROS2 & middleware
      Node & lifecycle model
      Executors & callback groups
      Topics / services / actions
      QoS profiles
      DDS / RMW
      iceoryx zero-copy
      colcon / ament / launch
      TF2 transform tree
      ROS2 testing
      rosbag2
      ros2_control
      CI/CD for robotics
      Multi-machine ROS2
      State machines in robotics
    Real-time & embedded
      Determinism & jitter / WCET
      AVR register programming
      USART / timers / PWM / ADC / TWI
      Parallax Propeller cogs
      Watchdog timers
      Serial packet protocols
      CAN bus
      RTOS fundamentals
      FreeRTOS tasks & queues
    Control theory
      PID
      State-space & pole placement
      LQR
      Kalman filter
      LQG & separation principle
      Complementary filter
      MPC & virtual fixtures
    Kinematics & dynamics
      Frames & SE(3) / rotations
      Forward kinematics & DH
      Jacobian
      Inverse kinematics & DLS
      Lagrangian / Newton-Euler / Hamiltonian
      Floating-base & whole-body
      Trajectory optimization
      Contact modeling
    Motor control
      BLDC & FOC
      Clarke / Park transforms
      H-bridge & PWM drive
      Encoders & I2C mux
      Dynamixel protocol
      [[Motor Control]]
    Perception & ML
      OpenCV pipelines
      YOLOv8 detection
      SLAM / RTAB-Map
      Odometry vs localization vs SLAM
      Nav2 stack
      Behavior trees
      Motion planning algorithms
      Camera model & projection
      Point cloud processing
      LNN & PINN
      LLM tool calling
      Sensor fusion
    Compute & acceleration
      Jetson Orin / JetPack
      CUDA & CuPy
      Docker / Compose
      NixOS
      TensorRT / edge inference
    Simulation & tooling
      Isaac Lab / Sim
      MuJoCo / Gazebo
      acados / OCS2 / CasADi
      SymPy / scipy
      VR teleop pipeline
      Collada & mesh formats
    Surgical robotics
      Multi-arm laparoscopy
      Teleop & motion scaling
      Safety-critical architecture
      Force feedback / haptics
      Virtual fixtures
      Latency budgets
      IEC 62304 / regulatory
    SWE & math foundations
      Modern C++ / real-time-safe
      Linear algebra
      Probability & optimization
      Git / build systems
      DSA patterns (NeetCode)
      CI/CD for robotics
      Robotics toolchain
      Behavioral interview prep
```

---

## Full taxonomy

### 🟢 ROS2 & middleware
- [[ROS2 Executors]] — single vs multi-threaded, callback groups, spin model
- [[ROS2 QoS]] — reliability, durability, history/depth, deadline, liveliness
- [[ROS2 Node & Lifecycle]] — managed nodes, state transitions, composition
- [[ROS2 Comm Patterns]] — topics vs services vs actions
- [[ROS2 Node Design Patterns]] — canonical structure, mutex patterns, watchdog, QoS gotchas (with real code)
- [[DDS & RMW]] — discovery, the RMW abstraction, vendor differences
- [[iceoryx Zero-Copy]] — shared-memory transport, when DDS isn't fast enough
- [[colcon ament & launch]] — build system, CMake, launch composition
- [[TF2]] — transform tree, lookup, broadcaster, /tf vs /tf_static, debugging
- [[ROS2 Testing]] — gtest unit tests, integration tests, launch tests, colcon test
- [[rosbag2]] — record/playback, hardware debugging workflow, filtering
- [[ros2_control]] — hardware abstraction layer, controllers, hardware interface plugin
- [[CI_CD for Robotics]] — GitHub Actions for ROS2, Docker-based CI, what to test
- [[Multi-Machine ROS2]] — DDS discovery, ROS_DOMAIN_ID, network config, bandwidth
- [[State Machines in Robotics]] — FSM, HSM, BT vs FSM, ROS2 lifecycle, common bugs

### 🟢 Real-time & embedded
- [[Real-Time Determinism]] — jitter, WCET, RT_PREEMPT, priority inversion
- [[RTOS Fundamentals]] — preemptive scheduling, priority inversion, semaphore vs mutex, RMS schedulability
- [[FreeRTOS]] — tasks, queues, semaphores, mutexes, ISR-safe API, Arduino Mega relevance
- [[AVR Register Programming]] — DDRx/PORTx/PINx, direct register I/O
- [[AVR Peripherals]] — USART (UBRR), Timers (overflow/prescaler), PWM, ADC, TWI (TWBR)
- [[Parallax Propeller]] — 8-cog model, dira/outa/ina, cog_run
- [[Watchdog Timers]] — WDT, the timeout failure mode I debugged
- [[Serial Packet Protocols]] — TankStatus 16-byte, odometry 22-byte, framing
- [[CAN Bus]] — arbitration, message IDs, SocketCAN Linux API, motor command sequence
- [[Dynamixel ROS2]] — Protocol 2.0, control table, bulk read/write, setup sequence

### 🟢 Control theory
- [[PID Control]] — terms, tuning, windup, derivative kick
- [[State-Space & Pole Placement]] — controllability, observability, eigenvalue placement
- [[LQR]] — cost J, Q/R weighting, Riccati, gain K
- [[Kalman Filter]] — predict/update, discrete form, process/measurement noise
- [[LQG]] — LQR + Kalman, separation principle
- [[Complementary Filter]] — sensor fusion for attitude, why I used it on the gimbal
- [[MPC & Virtual Fixtures]] — receding horizon, constraints, QP, the Kim research direction
- [[Frequency domain Analysis]] ----- what chbat thought us.
### 🟢 Kinematics & dynamics
- [[Frames & Rotations]] — SE(3), euler/quaternion/axis-angle, transform chains
- [[Forward Kinematics]] — DH parameters, transform composition
- [[Jacobian]] — velocity mapping, singularities, manipulability
- [[Inverse Kinematics & DLS]] — analytical vs numerical, damping λ
- [[Robot Dynamics Formulations]] — Lagrangian, Newton-Euler, Hamiltonian
- [[Floating-Base & Whole-Body]] — under-actuation, whole-body kinematics
- [[Trajectory Optimization]] — direct methods, collocation
- [[Contact Modeling]] — force thresholds, rising-edge detection

### 🟣 Perception & ML for robotics
- [[OpenCV Pipelines]] — classical CV, color/feature pipelines
- [[YOLOv8 Detection]] — semantic nav primitives, inference on edge
- [[SLAM & RTAB-Map]] — factor graphs, loop closure (summer queue)
- [[Odometry vs Localization vs SLAM]] — the three concepts, LiDAR-IMU fusion rationale, drift handling, what breaks in TF
- [[Behavior Trees]] — node types, Sequence/Fallback, Nav2 BT, BT vs FSM
- [[Motion Planning Algorithms]] — A*, Dijkstra, RRT, RRT*, PRM, C-space, DWB local planner
- [[Camera Model & Projection]] — pinhole model, intrinsics K, extrinsics, distortion, calibration, backprojection
- [[Point Cloud Processing]] — voxel grid, ICP, KD-tree, PCL, RTAB-Map pipeline
- [[Nav2 Stack]] — costmaps, planners, behavior trees, AMCL particle filter
- [[Home Service Bot]] — AMCL, Nav2, Kalman odometry, pick-and-place, full autonomy stack
- [[Lagrangian Neural Networks]] — structure-preserving ML, the bugs I fixed
- [[PINN Contact Estimation]] — physics-informed force sensing
- [[LLM Tool Calling]] — LLaMA 3.2, natural-language command parsing
- [[Sensor Fusion]] — combining IMU/encoder/vision

### 🟣 Compute & acceleration
- [[Jetson Orin Setup]] — JetPack, SDK Manager, the Nano
- [[CUDA & CuPy]] — RawKernel, GPU offload patterns
- [[Docker for ROS]] — Compose, ros:humble + ollama/ollama
- [[NixOS for Jetson]] — nixjetson, reproducible embedded
- [[Edge Inference]] — TensorRT, jetson_inference, CNN deployment

### 🟣 Simulation & tooling
- [[Isaac Lab]] — sim setup, UDP bridge, RigidObjectCfg
- [[MuJoCo & Gazebo]] — contact sim, when to use which
- [[Simulation Environments Comparison]] — Isaac vs MuJoCo vs Gazebo decision tree
- [[URDF & CAD Pipeline]] — manual URDF, fusion2urdf, loading into Isaac/Gazebo/MuJoCo
- [[Collada & Mesh Formats]] — .dae vs .stl, visual vs collision mesh, CAD export workflow, scale gotchas
- [[PX4 SITL & Drone Stack]] — MAVLink, OFFBOARD mode, SITL setup, NED/ENU frames
- [[acados OCS2 CasADi]] — optimization/MPC toolchains
- [[SymPy & scipy]] — Lagrangian derivation, solve_ivp
- [[VR Teleop Pipeline]] — the FENCE-BOT vr_pose → controller → sim chain

### 🟠 Surgical robotics domain (Articulus Surgical expertise!)

- [[Multi-Arm Laparoscopy]] — the Articulus system architecture
- [[Teleoperation & Motion Scaling]] — master-slave, tremor filtering
- [[Safety-Critical Architecture]] — fault handling, watchdogs, fail-safe states
- [[Force Feedback & Haptics]] — sensing, rendering, stability
- [[Virtual Fixtures]] — active constraints, the MPC project framing
- [[Latency Budgets]] — end-to-end timing, why zero-copy mattered
- [[Regulatory Context]] — ISO 62304,60601,13485 , FDA, what RSE roles expect you to know

### ⚪ SWE & math foundations
- [[Modern C++ for Robotics]] — RAII, real-time-safe code, no-alloc paths
- [[Programming Languages for Robotics]] — C++ vs Python vs MATLAB, when to use each
- [[Linear Algebra Refresher]] — the bits that show up in IK/control
- [[Probability & Optimization]] — for estimation and MPC
- [[Git & Build Systems]] — workflow, CMake depth
- [[Linux, IPC & System Design]] — process/thread model, IPC, concurrency, system design patterns
- [[DSA Patterns]] — NeetCode 150 in C++, pattern index + full templates (BFS, sliding window, heap, Dijkstra, binary search)
- [[CI_CD for Robotics]] — GitHub Actions, Docker ROS2 image, colcon CI, what to automate
- [[Robotics Toolchain]] — ros2 CLI, RViz2, rqt, GDB, perf, jtop, colcon, debugging workflow
- [[Behavioral Interview Prep]] — STAR method, project one-liners, WALL-E/FENCE-BOT stories, common RSE questions
- [[Interview Problem-Solving Framework]] — the 5-step framework, spoken scripts for every pattern, what to say when stuck
- [[Design_Patterns]] ---- The only design patterns you need to master for tech interview. gonna read this up every night until i got everything drilled (30 mins a day)
	
### 🔧 Projects & applied work
- [[FENCE-BOT]] — VR→UDP→ROS2→Isaac Lab IK, DLS, PINN contact, CSV logging (actual code)
- [[WALL-E V3]] — tracked robot: state machine, LLaMA NL control, YOLO nav, 3 serial buses (actual code)
- [[PX4 ArUco Spiral Project]] — spiral trajectory, ArUco precision landing, MAVSDK async (actual code)
- [[Foundations of Robotics — UR10e Project]] — FK/IK position+velocity, PD control, Jacobian, MuJoCo API (NYU final project)
- [[Home Service Bot]] — full autonomy stack: AMCL, Nav2, KF odometry, pick-and-place
- [[PX4 SITL & Drone Stack]] — drone autopilot, MAVLink, SITL setup
- [[SAE BAJA & Engineering Design]] — SolidWorks design, DFMEA, suspension, vehicle systems
- [[VR Teleop Pipeline]] — FENCE-BOT end-to-end: VR → ROS2 → Isaac Lab
- 

---

## Progress tracker

```dataview
TABLE status, domain, last-reviewed
FROM "Robotics_Knowledge_Vault"
WHERE type = "concept"
SORT status ASC
```
*(Requires the Dataview plugin — auto-lists every note and its drill status.)*
