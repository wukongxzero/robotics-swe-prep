---
type: concept
domain: Perception & ML
status: drafted
last-reviewed:
tags: [perception, slam, rtabmap, summer-queue]
---

# SLAM & RTAB-Map

> [!warning] Summer queue — conceptual, not yet shipped
> RTAB-Map SLAM was in the WALL-E V3 proposal but **deferred to summer** — not implemented yet. Frame this honestly: "solid conceptual grasp, building it this summer." You have built a home service bot before so the practical intuition is there.


---

## Core idea

SLAM = **Simultaneous Localization and Mapping**: build a map of an unknown environment _while_ simultaneously tracking your pose within it. It's circular — you need a map to localize, and a pose to map — so it's posed as a **joint estimation problem** solved incrementally. Sensor noise means pose error and map error feed each other; loop closure + global optimization keep it bounded.

## Key facts

### Front-end vs back-end

**Front-end** (runs every frame, must be fast):
- Processes raw sensor data (RGB-D, LiDAR, stereo)
- Extracts features or matches point clouds
- Estimates frame-to-frame motion (visual/lidar odometry)
- Does data association — which observation matches which landmark
- Produces a stream of pose constraints for the back-end

**Back-end** (runs on loop closure or periodically):
- Maintains a **factor graph** — nodes = robot poses + landmarks, edges = constraints (odometry, observations, loop closures)
- When loop closure fires, adds an edge between current pose and the revisited one
- Runs nonlinear least-squares optimization (g2o, GTSAM, Ceres) over the whole graph
- Produces a globally consistent trajectory and map

### Loop closure
The robot recognizes it has **returned to a previously visited place** and adds a constraint that corrects all the accumulated drift since the last visit. Without loop closure, odometry error grows unbounded and the map warps into an unusable shape. This is the hardest and most important part of SLAM.

**How RTAB-Map detects it**: bag-of-words on visual features — each image is represented as a histogram of visual words; similarity search over the memory finds candidate matches; geometric verification confirms the loop.

### Why factor graphs over EKF-SLAM
- EKF-SLAM: covariance matrix grows quadratically with landmark count → doesn't scale past ~100 landmarks
- Factor graphs exploit **sparsity** — most poses only connect to nearby poses/landmarks; can optimize large maps efficiently
- Modern standard: g2o, GTSAM, iSAM2. EKF-SLAM is mostly historical now.

### RTAB-Map specifically
- **Real-Time Appearance-Based Mapping**
- Graph-based SLAM with appearance-based loop closure (bag-of-words)
- **Memory management**: limits the working memory to keep loop closure detection real-time on large maps — older nodes moved to long-term memory
- Sensors: RGB-D (D435, D415), stereo, LiDAR, or fusion of RGB-D + LiDAR
- ROS2 package: `rtabmap_ros` — publishes `/map`, `/odom`, `/tf`; subscribes to camera + depth topics
- Feeds into [[Nav2 Stack]] — SLAM gives the map and localization; Nav2 uses it for planning

### SLAM modes
- **Mapping mode**: building the map while moving
- **Localization mode**: map is fixed, robot localizes within it (like AMCL but RTAB-Map's version)
- **Lifelong SLAM**: continuous mapping + localization in a changing environment

### Typical ROS2 integration
```
D435 camera → /camera/depth, /camera/color
        ↓
rtabmap_ros node
        ↓
/map (OccupancyGrid)    → Nav2 costmaps
/odom (corrected)       → TF tree
/rtabmap/mapData        → 3D point cloud map
```


## Links
- Related: [[Nav2 Stack]], [[Kalman Filter]], [[Sensor Fusion]], [[OpenCV Pipelines]], [[Jetson Orin Setup]]
- Parent: [[00 Knowledge Map]]

---

