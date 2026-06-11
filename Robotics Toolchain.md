---
type: concept
domain: SWE & math foundations
status: drafted
last-reviewed: 2026-06-02
tags: [tools, debugging, profiling, workflow, ros2, jetson]
---

# Robotics Toolchain

> [!question] Explain it cold
> - What tools do you use to debug a ROS2 node that isn't receiving messages?
> - How do you profile a C++ node that's missing its deadline?
> - What's your workflow from code to robot?

---

## ROS2 CLI — daily debugging

```bash
# Node inspection
ros2 node list                          # all running nodes
ros2 node info /node_name              # pubs, subs, services, params

# Topic inspection
ros2 topic list                         # all topics
ros2 topic info /topic --verbose        # QoS, publisher/subscriber count
ros2 topic echo /topic                  # print messages
ros2 topic hz /topic                    # message rate
ros2 topic bw /topic                    # bandwidth

# TF debugging
ros2 run tf2_ros tf2_echo odom base_footprint
ros2 run tf2_tools view_frames         # outputs frames.pdf
ros2 run tf2_ros tf2_monitor           # continuous TF health

# Parameters
ros2 param list /node_name
ros2 param get /node_name param_name
ros2 param set /node_name param_name value

# Services & actions
ros2 service list
ros2 action list
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose "{...}"

# Bags
ros2 bag record -a -o my_bag
ros2 bag play my_bag
ros2 bag info my_bag
```

---

## RViz2 — visualization

| Display | What to add it for |
|---------|-------------------|
| Map | `/map` — RTAB-Map occupancy grid |
| LaserScan | `/scan` — depth→laserscan output |
| PointCloud2 | `/cloud_map` — 3D map |
| TF | All coordinate frames |
| Odometry | `/odom` — robot path |
| Camera | `/camera/color/image_raw` |
| MarkerArray | Custom visualization from nodes |

**Fixed frame:** set to `map` for navigation, `odom` for local debugging.

---

## rqt — GUI toolbox

```bash
rqt_graph          # visualize node/topic connections
rqt_plot           # live plot of numeric topics (e.g. /odom pose)
rqt_console        # filter and view ros2 logs
rqt_image_view     # display image topics
rqt               # launch rqt with all plugins
```

`rqt_graph` is the first tool to open when a topic isn't arriving — shows exactly who's publishing and who's subscribing.

---

## Profiling C++ nodes

```bash
# CPU profiling with perf
perf record -g ros2 run wall_e_bringup mega_node
perf report

# Valgrind (memory leaks — use on dev machine, too slow for robot)
valgrind --leak-check=full ros2 run wall_e_bringup mega_node

# GDB for crashes
gdb -ex run --args ros2 run wall_e_bringup mega_node
# When crash: bt (backtrace), p variable_name, frame N

# ROS2 tracing (timing, latency)
ros2 run tracetools_launch trace \
  --session-name my_trace \
  -- ros2 launch wall_e_bringup wall_e.launch.py
```

---

## Jetson-specific tools

```bash
# GPU/CPU monitoring
jtop                          # like htop but Jetson-aware (install: pip install jetson-stats)
tegrastats                    # built-in: CPU/GPU/RAM/temp

# Power mode (performance vs efficiency)
sudo nvpmodel -m 0            # MAXN (max performance)
sudo nvpmodel -m 1            # 15W
sudo jetson_clocks            # lock clocks to max

# Camera debugging
rs-enumerate-devices          # list RealSense devices
realsense-viewer              # GUI viewer for D435
```

---

## Build tools

```bash
# colcon
colcon build --packages-select wall_e_bringup        # build one package
colcon build --symlink-install                       # symlinks instead of copy (faster iteration)
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Debug   # debug symbols
colcon test --packages-select wall_e_bringup         # run tests
colcon test-result --verbose                         # see test results

# CMake
cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo ..           # release + debug info
make -j$(nproc)                                      # parallel build

# rosdep (dependency management)
rosdep install --from-paths src --ignore-src -r -y   # install all deps
```

---

## Git workflow for robotics

```bash
# Feature branch per subsystem
git checkout -b feature/slam-integration
git checkout -b fix/mega-node-thread-safety

# Useful aliases
git log --oneline --graph --all   # visual branch history
git bisect start                  # binary search for regression
git stash                         # save work before switching context

# Before pushing hardware code: always test in sim first
```

---

## Debugging workflow — systematic approach

```
1. ros2 topic hz /topic      → is data arriving? at correct rate?
2. ros2 topic info /topic -v → QoS mismatch? (most common silent failure)
3. rqt_graph                 → is publisher connected to subscriber?
4. ros2 run tf2_tools view_frames → TF chain broken?
5. ros2 node info /node      → is node running? correct params?
6. ros2 bag record + replay  → capture the failure for offline analysis
7. gdb / perf                → crash or performance issue in C++
```

---

## Links
- Related: [[ROS2 Node Design Patterns]], [[rosbag2]], [[TF2]], [[CI_CD for Robotics]], [[Jetson Orin Setup]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

First tool to open when a ROS2 topic isn't arriving? ? `rqt_graph` to see if publisher and subscriber are connected, then `ros2 topic info /topic --verbose` to check QoS compatibility. QoS mismatch is the most common silent failure — no error, no data.

How do you check the rate and bandwidth of a topic? ? `ros2 topic hz /topic` for message rate. `ros2 topic bw /topic` for bandwidth in bytes/sec. If rate is 0, the publisher stopped or QoS is mismatched.

Jetson CPU/GPU monitoring command? ? `jtop` (install with `pip install jetson-stats`) — shows CPU cores, GPU, RAM, temperature, and power mode. `tegrastats` is the built-in alternative.
