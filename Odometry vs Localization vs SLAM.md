---
type: concept
domain: Perception & ML
status: reviewed
last-reviewed: 2026-06-10
tags: [odometry, localization, slam, lidar, imu, drift, navigation]
---

# Odometry vs Localization vs SLAM


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

