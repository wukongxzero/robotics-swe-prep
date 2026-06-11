---
type: concept
domain: ROS2 & middleware
status: drafted
last-reviewed: 2026-06-02
tags: [ros2, debugging, rosbag2, recording, playback]
---

# rosbag2


---

## Core idea

rosbag2 records ROS2 topics to disk and plays them back later. Essential for hardware debugging: record a run on the robot, replay it on your laptop, inspect what went wrong without the robot present. Also used for dataset collection and regression testing.

---

## Key commands

```bash
# Record all topics
ros2 bag record -a -o my_bag

# Record specific topics only (recommended — bags get large fast)
ros2 bag record /odom /scan /camera/color/image_raw /cmd_vel -o my_bag

# Play back
ros2 bag play my_bag

# Play at half speed
ros2 bag play my_bag --rate 0.5

# Play and loop
ros2 bag play my_bag --loop

# Inspect bag metadata
ros2 bag info my_bag

# List topics in bag
ros2 bag info my_bag | grep Topic
```

---

## Hardware debugging workflow

```
1. SSH into Jetson, launch the robot stack
2. ros2 bag record /odom /scan /wall_e/camera/image_raw /wall_e/state -o debug_run
3. Drive robot, reproduce the failure
4. Ctrl+C recording
5. scp the bag to laptop
6. ros2 bag play debug_run  ← replay on laptop
7. Open RViz, inspect what the robot saw
8. Fix the bug without needing the robot
```

---

## Recording for WALL-E

```bash
# Useful topics to always record
ros2 bag record \
  /odom \
  /scan \
  /map \
  /cmd_vel \
  /nav_cmd_vel \
  /wall_e/state \
  /wall_e/camera/image_raw \
  /rtabmap/odom \
  -o wall_e_run_$(date +%Y%m%d_%H%M%S)
```

---

## Bag file format

- Default: SQLite3 database (`.db3` file + `metadata.yaml`)
- Can convert to MCAP (better for large sensor data)
- Each message stored with timestamp, topic, serialized data

---

## Filtering and converting

```bash
# Convert to MCAP format
ros2 bag convert -i my_bag -o my_bag_mcap --output-format mcap

# Play only specific topics from a bag
ros2 bag play my_bag --topics /odom /scan

# Start playback at a specific time offset
ros2 bag play my_bag --start-offset 30.0  # skip first 30 seconds
```

---

## Gotchas

| Gotcha | Fix |
|--------|-----|
| Bag grows to GB on camera topics | Record compressed: `--compression-mode file` |
| Clock issues on playback | Add `--clock` flag to publish `/clock` from bag |
| Nodes using sim_time don't play well | Run with `--clock`, set `use_sim_time: true` on nodes |
| Large images slow playback | Record at lower rate: use `--max-cache-size` |

---

## Links
- Related: [[ROS2 Node Design Patterns]], [[WALL-E V3]], [[Real-Time Determinism]]
- Parent: [[00 Knowledge Map]]

---

