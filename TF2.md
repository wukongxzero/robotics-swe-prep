---
type: concept
domain: ROS2 & middleware
status: drafted
last-reviewed: 2026-06-02
tags: [ros2, tf2, transforms, frames, navigation]
---

# TF2


---

## Core idea

TF2 maintains a tree of coordinate frame transforms over time. Any node can query "where is frame B relative to frame A at time T?" without knowing who published that transform. Essential for navigation, manipulation, and sensor fusion — everyone agrees on where things are in space.

---

## The transform tree for WALL-E

```
map
 └── odom              ← published by RTAB-Map (map→odom correction)
      └── base_footprint  ← published by odom_to_tf / diff-drive
           └── base_link  ← published by robot_state_publisher
                ├── camera_link
                │    ├── camera_depth_optical_frame
                │    └── camera_color_optical_frame
                ├── left_track
                ├── right_track
                └── imu_link
```

If any link in this chain is missing, Nav2/RTAB-Map will fail silently or with a TF timeout error.

---

## Key API

```python
import rclpy
from rclpy.node import Node
from tf2_ros import Buffer, TransformListener
from geometry_msgs.msg import PointStamped
import tf2_geometry_msgs  # required for do_transform_point

class MyNode(Node):
    def __init__(self):
        super().__init__('my_node')
        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)

    def get_transform(self):
        try:
            # Get latest available transform
            t = self.tf_buffer.lookup_transform(
                'map',           # target frame
                'base_footprint', # source frame
                rclpy.time.Time()  # latest available
            )
            return t
        except Exception as e:
            self.get_logger().warn(f'TF lookup failed: {e}')
            return None

    def transform_point(self, point_stamped):
        # Transform a point from camera frame to map frame
        return self.tf_buffer.transform(point_stamped, 'map', timeout=rclpy.duration.Duration(seconds=0.1))
```

```cpp
// C++ version
#include <tf2_ros/buffer.h>
#include <tf2_ros/transform_listener.h>
#include <tf2_ros/transform_broadcaster.h>

// In node constructor:
tf_buffer_ = std::make_shared<tf2_ros::Buffer>(this->get_clock());
tf_listener_ = std::make_shared<tf2_ros::TransformListener>(*tf_buffer_);

// Lookup:
auto t = tf_buffer_->lookupTransform("map", "base_footprint",
    tf2::TimePointZero);  // latest available

// Broadcast a transform:
tf2_ros::TransformBroadcaster br(this);
geometry_msgs::msg::TransformStamped ts;
ts.header.stamp = now();
ts.header.frame_id = "odom";
ts.child_frame_id = "base_footprint";
ts.transform.translation.x = x;
ts.transform.rotation.w = 1.0;
br.sendTransform(ts);
```

---

## /tf vs /tf_static

| | /tf | /tf_static |
|--|-----|------------|
| Use for | Moving transforms (odom→base) | Fixed transforms (base→camera) |
| Latched | No — expires from buffer | Yes — late subscribers get it |
| Broadcaster | `TransformBroadcaster` | `StaticTransformBroadcaster` |
| Update rate | Every control cycle | Once at startup |

**Rule:** if the transform never changes (sensor mount, camera optical frame), use `/tf_static`. If it changes (robot pose, joint angles), use `/tf`.

---

## Debugging TF

```bash
# Print the full TF tree
ros2 run tf2_tools view_frames

# Echo a specific transform
ros2 run tf2_ros tf2_echo odom base_footprint

# Check if a transform exists
ros2 topic echo /tf --once | grep frame_id

# Find broken TF chain
ros2 run tf2_ros tf2_monitor
```

---

## Common failures

| Error | Cause | Fix |
|-------|-------|-----|
| `Transform timeout` | Frame not being published | Check who publishes it, add missing broadcaster |
| `Frame does not exist` | Wrong frame name, typo | `ros2 run tf2_tools view_frames` to see actual names |
| `Extrapolation into the past` | Buffer too short, clock jump | Increase buffer duration or use `TimePointZero` |
| `No common time` | Nodes using different clocks | Ensure all use same `use_sim_time` setting |

---

## Where transforms come from in WALL-E

| Transform | Publisher |
|-----------|-----------|
| `map → odom` | RTAB-Map (loop closure correction) |
| `odom → base_footprint` | `odom_to_tf.py` (from /odom) |
| `base_footprint → base_link` | `robot_state_publisher` (URDF fixed joint) |
| `base_link → camera_link` | `robot_state_publisher` (URDF fixed joint) |
| `base_link → left/right_track` | `robot_state_publisher` (from /joint_states) |

---

## Links
- Related: [[ROS2 Node Design Patterns]], [[WALL-E V3]], [[SLAM & RTAB-Map]], [[Nav2 Stack]]
- Parent: [[00 Knowledge Map]]

---

