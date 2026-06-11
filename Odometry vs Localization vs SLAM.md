---
type: concept
domain: Perception & ML
status: reviewed
last-reviewed: 2026-06-10
tags: [odometry, localization, slam, lidar, imu, drift, navigation]
---

# Odometry vs Localization vs SLAM

> [!question] Explain it cold
> - What is the difference between odometry, localization, and SLAM?
> - Why is LiDAR fused with IMU instead of used alone?
> - How is odometry drift handled in practice?

---

## The three concepts — one line each

| Concept | What it does | What it needs |
|---------|-------------|---------------|
| **Odometry** | Estimates how far you've moved from a known start | Only the robot's own sensors (encoders, IMU) |
| **Localization** | Estimates your pose within a **known, existing map** | A pre-built map + sensor observations |
| **SLAM** | Builds the map and localizes within it **simultaneously** | No prior map — works in unknown environments |

---

## Odometry

Dead reckoning from incremental sensor data. You integrate wheel encoder ticks (or IMU angular velocity) to estimate displacement from the start position.

```
pose(t) = pose(t-1) + Δpose_from_encoders
```

**Problem: drift.** Errors accumulate with every step. After 10 metres, position error might be 10–30cm. After 100 metres, it's unusable.

**Sources of drift:**
- Wheel slip (terrain, acceleration)
- Encoder resolution limits
- IMU bias (gyro integrates to heading error over time)
- Uneven ground (wheel diameter changes)

Odometry gives the `odom → base_footprint` TF. It is always drifting. It is never corrected by itself.

---

## Localization (given a map)

You have a map. You use sensor observations to estimate where you are in it.

**AMCL (Nav2 default):** particle filter. Maintains N hypotheses of where you might be. Each particle is a pose. Sensor scan is compared to the map — particles that match get higher weight. Particles are resampled toward high-weight regions.

**RTAB-Map localization mode:** uses visual features and scan matching against stored nodes.

Localization gives the `map → odom` TF — it corrects the drift in odometry. The odom frame drifts; the map frame is the ground truth.

```
map → odom (published by AMCL or RTAB-Map)  ← drift correction
odom → base_footprint (published by encoders)  ← raw odometry
```

**Problem:** requires a pre-built map. If the environment changes, localization degrades.

---

## SLAM

No map. Builds it on the fly while localizing. The chicken-and-egg problem: you need a map to localize, and a pose to map.

**Solution:** joint probabilistic estimation — maintain uncertainty over both map and pose simultaneously. Loop closure is the key mechanism that keeps error bounded.

```
Without loop closure: error grows unbounded (open loop)
With loop closure:    recognise a revisited place → add constraint → 
                      optimise the full trajectory → error shrinks
```

SLAM gives both `map → odom` TF AND the map itself. It is the most general solution — works in unknown environments.

---

## Why LiDAR + IMU fusion?

**LiDAR alone:**
- Accurate scan matching (cm-level)
- But slow: 10–20 Hz scan rate
- Between scans: no motion estimate → fast motion = missed frames, bad matching

**IMU alone:**
- 200–1000 Hz measurement rate
- Great for short-term motion (rotation, acceleration)
- But integrates to velocity → position → error grows fast (drift)

**Fused (LiDAR-IMU odometry, e.g. LOAM, LIO-SAM, FAST-LIO):**
- IMU pre-integration fills the gap between LiDAR scans
- LiDAR scan matching corrects IMU drift at each scan
- Result: high-rate, low-drift pose estimate

```
IMU @ 200Hz → pre-integrate poses between scans
LiDAR @ 10Hz → scan match → correct IMU drift
Output: pose at 200Hz with LiDAR accuracy
```

**In WALL-E context:** RealSense D435 depth camera fills the LiDAR role. IMU (MPU-6050 from Uno) could be added to RTAB-Map for tighter odometry — set `subscribe_imu: true`.

---

## How odometry drift is handled

| Method                     | How it works                                             | Where used              |
| -------------------------- | -------------------------------------------------------- | ----------------------- |
| **Loop closure**           | Detect revisited place → add constraint → optimise graph | RTAB-Map, ORB-SLAM      |
| **EKF/UKF**                | Fuse encoder odom + IMU → reduce per-step error          | robot_localization node |
| **Scan matching**          | Align current scan to previous → correct heading         | SLAM front-end          |
| **GPS correction**         | Absolute position fix when outdoor                       | Mobile robots outdoors  |
| **Map-based localization** | AMCL re-estimates pose from map                          | Post-mapping operation  |

**In practice for WALL-E hardware:**
1. RTAB-Map does scan matching + loop closure → corrects long-term drift
2. EKF (robot_localization) would fuse wheel odom + IMU → reduces short-term drift between RTAB-Map updates
3. Result: `map → odom` stays accurate, Nav2 plans correctly

---

## What breaks if TF is wrong

| Broken transform                | Symptom                                                               |
| ------------------------------- | --------------------------------------------------------------------- |
| `odom → base_footprint` missing | RTAB-Map can't track motion, map never builds                         |
| `map → odom` missing/stale      | Nav2 thinks robot is at wrong position, plans wrong path              |
| `base_link → camera_link` wrong | RTAB-Map maps features at wrong 3D position, map distorted            |
| `odom → base_footprint` drifted | Robot gradually drives off-course (expected — RTAB-Map corrects this) |
| Frame name mismatch (typo)      | `Extrapolation into the past` or `Frame does not exist` errors        |

→ See [[TF2]] for full debugging tools.

---

## Links
- Related: [[SLAM & RTAB-Map]], [[Sensor Fusion]], [[Kalman Filter]], [[TF2]], [[Nav2 Stack]], [[WALL-E V3]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

One-line difference: odometry vs localization vs SLAM? ? Odometry: how far you've moved (encoder integration, drifts). Localization: where you are in a known map (AMCL, scan matching). SLAM: build the map AND localize simultaneously, no prior map needed.

Why fuse LiDAR with IMU instead of using LiDAR alone? ? LiDAR scans at 10–20Hz — fast motion between scans causes bad matching. IMU runs at 200–1000Hz and pre-integrates pose between scans. LiDAR corrects IMU drift at each scan. Result: high-rate accurate pose estimate that LiDAR or IMU alone can't achieve.

How does loop closure handle odometry drift in SLAM? ? When the robot revisits a known place, SLAM detects the match and adds a constraint between the current pose and the stored one. A global graph optimisation then adjusts the entire trajectory to be consistent. Error that accumulated in the open loop is redistributed and reduced.

What publishes `map → odom` TF and what publishes `odom → base_footprint`? ? `map → odom`: RTAB-Map or AMCL — the drift correction from localization/SLAM. `odom → base_footprint`: raw encoder odometry (mega_node, diff-drive plugin, or odom_to_tf.py). The two stack together to give the robot's pose in the map frame.

How does EKF in robot_localization reduce odometry drift? ? Fuses wheel odometry (drifts from slip) and IMU (drifts from bias). Filter weights both by noise covariances Q and R. Combined estimate drifts slower than either sensor alone. Output: pose estimate (position + orientation + velocity).

What does scan matching do in SLAM and how does it differ from loop closure? ? Scan matching continuously aligns the current scan against the previous one on every frame — small corrections at high frequency. Loop closure is a large periodic correction when a known place is revisited. Scan matching fights short-term drift; loop closure bounds long-term error.

## Drill notes (2026-06-10)
- Concepts understood from day one. Consistent gap: stopping after "what" without explaining the "how" (mechanism).
- Loop closure: needed multiple prompts to reach graph optimisation → constraint → trajectory adjustment → error redistribution.
- LiDAR-IMU: knew failure mode but needed prompting to explain pre-integration.
- EKF: knew Jacobians vs sigma points but initially couldn't connect EKF to drift correction or state the output.
- All three locked by end of session after re-drilling.
