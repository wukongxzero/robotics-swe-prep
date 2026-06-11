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

> [!question] Explain it cold
> *Answer from memory first.*
>
> - What problem does SLAM solve, and why is it circular?
> - Front-end vs back-end of a modern SLAM system.
> - What is loop closure and why does it matter?

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


## Interview follow-ups
- **Q:** Why is SLAM hard / circular?
  - **A:** You need a map to localize and a pose to build the map — they're coupled. Noise in one corrupts the other. Loop closure + global optimization break the cycle by periodically correcting the whole trajectory.
- **Q:** Front-end vs back-end?
  - **A:** Front-end does perception, odometry, and data association every frame. Back-end optimizes the pose/factor graph for global consistency — runs on loop closure or periodically.
- **Q:** What is loop closure and why does it matter?
  - **A:** Recognizing a previously-visited place and adding a graph constraint that corrects all accumulated drift since. Without it, the map warps unboundedly.
- **Q:** Why factor graphs over EKF-SLAM?
  - **A:** EKF-SLAM covariance grows quadratically with landmarks — doesn't scale. Factor graphs exploit sparsity, handle large maps, and re-linearize efficiently. Modern SLAM is entirely factor-graph based.
- **Q:** What does RTAB-Map's memory management do?
  - **A:** Keeps the working memory bounded so loop closure search stays real-time. Older nodes are moved to long-term memory and only recalled when a candidate loop closure is near them.


## Links
- Related: [[Nav2 Stack]], [[Kalman Filter]], [[Sensor Fusion]], [[OpenCV Pipelines]], [[Jetson Orin Setup]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does SLAM stand for and what does it solve?
?
Simultaneous Localization and Mapping — build a map of an unknown environment while tracking pose in it, as a joint estimation problem.

Why is SLAM called circular?
?
You need a map to localize and a pose to map — they depend on each other. Noise in one corrupts the other.

What is loop closure?
?
Recognizing a previously-visited place and adding a constraint that corrects all accumulated drift since the last visit. Without it, the map warps unboundedly.

Front-end vs back-end of SLAM?
?
Front-end: perception, odometry, data association — runs every frame. Back-end: optimizes the factor graph for global consistency — runs on loop closure.

What does RTAB-Map use for loop closure detection?
?
Bag-of-words on visual features — images are histograms of visual words; similarity search finds candidates; geometric verification confirms.

Why did factor graphs replace EKF-SLAM?
?
EKF-SLAM covariance grows quadratically with landmarks and doesn't scale. Factor graphs exploit sparsity, handle large maps, and re-linearize efficiently.

What is RTAB-Map's memory management for?
?
Keeps the working memory bounded so loop closure search stays real-time. Older nodes move to long-term memory.
