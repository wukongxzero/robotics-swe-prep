---

## type: concept domain: Simulation & tooling status: drafted last-reviewed: tags: [sim, teleop, vr, ros2, fence-bot, deep, differentiator]

# VR Teleop Pipeline

> [!star] Your end-to-end systems-integration story This is the FENCE-BOT note that proves you can build a _complete real-time pipeline_ across VR, ROS2 C++, IK, and a GPU sim — not just one component. Systems integration is exactly what generalist RSE roles screen for. Know the data path and the bugs cold.

> [!question] Explain it cold
> 
> - Trace the data path from VR controller to simulated arm motion.
> - What runs at what rate, and why?
> - What were the integration bugs and how did you find them?

---

## Core idea

The FENCE-BOT teleoperation pipeline maps a human's VR controller motion to a simulated 6-DOF arm in real time: VR pose → ROS2 middleware → IK → simulator. It's the surgical-teleop pattern in miniature (master device drives slave manipulator) and a complete real-time systems-integration build.

## The data path (know this cold)

```
VR controller
  → vr_udp_publisher        (ingest VR pose over UDP)
  → /vr_pose                (ROS2 topic)
  → robot_controller @ 50Hz (C++ middleware)
  → DLS IK (λ=0.1)          (pose → joint commands)
  → UDP → Isaac Lab :5006   (drive ASEM V2 arm)
  → contact sensing + CSV logging
```

- **`vr_udp_publisher`**: ingests the VR controller pose and publishes it as `/vr_pose`.
- **`robot_controller`** (C++, **50 Hz**): the core middleware node — consumes `/vr_pose`, runs [[Inverse Kinematics & DLS|DLS IK]] (λ=0.1) to turn the target pose into joint commands, sends them over UDP to [[Isaac Lab]] on **port 5006**.
- **50 Hz** because teleop needs to feel responsive but the IK + sim loop must keep up deterministically ([[Real-Time Determinism]]); too fast and you starve the solve, too slow and it lags the hand.
- **Contact sensing + CSV logging**: [[Contact Modeling|force-threshold + rising-edge]] hit detection, logging t/xyz/error/in_contact/force_mag for analysis and as [[Lagrangian Neural Networks|LNN]] training data.

## Where I've used it

- **FENCE-BOT**: built this whole chain from scratch in ROS2 C++. Validated end-to-end with a _fake_ VR UDP source first (fake VR → publisher → /vr_pose → controller → Isaac DLS IK → ASEM circle sweep) before real VR — a smart de-risking step. Red cube spawned via [[Isaac Lab|RigidObjectCfg]].
- **Integration bugs I resolved** (the war story): wrong source filename in `CMakeLists.txt`, missing runtime dependencies, running `colcon` from the wrong directory, **port 5005/5006 conflict**, and stale binaries from not re-sourcing ([[colcon ament & launch]]). Classic distributed-systems integration debugging.

## Interview follow-ups

- **Q:** Walk me through your teleop pipeline.
    - **A:** VR controller pose comes in over UDP to a publisher node, goes out on /vr_pose, my C++ robot_controller at 50 Hz runs DLS IK to convert the target pose to joint commands, and sends them over UDP to the arm in Isaac Lab. Contact is detected by force threshold with rising-edge logic and logged to CSV.
- **Q:** Why 50 Hz?
    - **A:** Responsive enough for teleop to feel direct, while keeping the IK + sim loop deterministic. Faster risks starving the solve; slower lags the operator's hand. It's the teleop responsiveness vs compute-budget tradeoff.
- **Q:** How did you de-risk the integration?
    - **A:** Validated end-to-end with a fake VR UDP source driving a known circle sweep before plugging in real VR — so I could isolate the ROS2/IK/sim chain from VR hardware issues. Most of the bugs were build/integration (CMake filename, deps, colcon dir, port conflict, stale binaries), found by bisecting the pipeline stage by stage.
- **Q:** How does this relate to surgical robotics?
    - **A:** It's the master-slave teleoperation pattern — operator device drives a manipulator with IK in between — which is exactly surgical teleop, where you'd add motion scaling, tremor filtering, and [[MPC & Virtual Fixtures|virtual fixtures]] on top.

## Gotchas / what trips me up

- Port conflicts (5005/5006) — the kind of integration bug that's invisible until two things fight for a port.
- Stale binaries / wrong colcon dir masking real logic — re-source and rebuild before debugging logic.
- Forgetting to validate with a synthetic source first — saves hours vs debugging VR + pipeline simultaneously.

## Links

- Related: [[Inverse Kinematics & DLS]], [[Isaac Lab]], [[Contact Modeling]], [[colcon ament & launch]], [[Real-Time Determinism]], [[Teleoperation & Motion Scaling]], [[MPC & Virtual Fixtures]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Trace the FENCE-BOT VR teleop data path. ? VR controller → vr_udp_publisher → /vr_pose topic → robot_controller (C++, 50Hz) → DLS IK (λ=0.1) → UDP to Isaac Lab port 5006 → drives ASEM V2 arm, with contact sensing + CSV logging.

Why run the teleop controller at 50 Hz? ? Responsive enough for teleop to feel direct while keeping the IK+sim loop deterministic — faster starves the solve, slower lags the operator's hand.

How did you de-risk the teleop integration? ? Validated end-to-end with a fake VR UDP source driving a known circle sweep before real VR, isolating the ROS2/IK/sim chain — then bisected stage-by-stage for the build/integration bugs (CMake filename, deps, colcon dir, port 5005/5006 conflict, stale binaries).

How does the VR teleop pipeline map to surgical robotics? ? It's the master-slave teleoperation pattern (operator device → IK → manipulator) — surgical teleop, where you'd add motion scaling, tremor filtering, and virtual fixtures on top.