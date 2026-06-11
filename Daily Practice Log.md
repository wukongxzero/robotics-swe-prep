---
type: log
domain: DSA & Robotics
status: active
last-reviewed:
tags: [dsa, robotics, practice, log]
---

# Daily Practice Log

Track every problem solved. One row per problem. Consistency > volume.

---

## DSA Problems

| Date | # | Problem | Approach | Time | Notes |
|------|---|---------|----------|------|-------|
| 2026-05-26 | 1 | Contains Duplicate | Brute force + hashmap | — | Python |
| 2026-05-26 | 2 | Two Sum | Brute force + hashmap | — | Python |
| 2026-05-29 | 3 | Valid Anagram | Sort + hashmap | — | C++ |
| 2026-05-29 | 4 | Group Anagrams | Sorted key hashmap | — | C++ |
| 2026-05-31 | 5 | Top K Frequent Elements | Hashmap + min-heap size k | — | C++ |
| 2026-05-31 | 6 | Encode & Decode Strings | Length-prefix `len#word` | — | C++ ✓ |

---

## Robotics Problems

| Date | Problem | Concept | Outcome | Notes |
|------|---------|---------|---------|-------|
| 2026-05-28 | FK 2D arm | Forward kinematics | Done | Python |
| 2026-05-28 | FK 3D arm + Jacobian + DLSIK | Kinematics | Done | Python |

---

## WALL-E Sim Progress

| Date | Work Done |
|------|-----------|
| 2026-06-02 | Full Gazebo sim stack working. URDF stable: sphere passive supports (pitch+turn fix), cylinder drive wheels, damping, high friction, lowered CoM. RTAB-Map with scan enabled. odom_to_tf bridge. Map builds. Loop closure weak in sim (featureless walls) — expected to work on hardware. Next: Jetson hardware bringup. |
| 2026-06-04 | Fixed RTAB-Map delay bug (DetectionRate 0→1Hz), camera 2Hz→5Hz, Nav2 transform_tolerance 1.5s. Fixed cmd_vel chain (controller was publishing to cmd_vel_nav not /cmd_vel). Added 4 colored landmark boxes to test_room.sdf — plain walls have zero visual features so RTAB-Map built an empty map. Robot successfully drove to Nav2 goal autonomously. Gazebo sim chapter CLOSED. Next: Isaac Sim port for RL training + perception coursework. Hardware chapter closed. |

---

## Drill Sessions

| Date | Notes Drilled | Weak Spots |
|------|--------------|------------|
| 2026-06-02 | ROS2 Node Design Patterns, ROS2 Executors, ROS2 QoS, ROS2 Comm Patterns, Modern C++ for Robotics | Mutex targets data not mutex; `new_data_` flag; destructor "why"; QoS compatibility direction; action internals (5 components); RAII definition; RT forbidden "why"; shared_ptr atomic refcount cost |
| 2026-06-04 | Kalman Filter (re-drill — weak spots from 2026-05-29) | GPS chain (R↑→K↓→trust model) needed scaffolding x2; K intuition ("chases less uncertain source") needed anchor; EKF/UKF distinction clean |

---

## Links
- Related: [[DSA Patterns]], [[Forward Kinematics]], [[Jacobian]], [[Inverse Kinematics & DLS]]
- Parent: [[00 Knowledge Map]]
