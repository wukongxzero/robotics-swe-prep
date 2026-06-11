---

## type: project domain: ROS2 & Middleware status: drafted last-reviewed: tags: [ros2, navigation, llama, yolo, serial, state-machine, cpp, python]

# WALL-E V3

> [!question] Explain it cold
>
> - What does WALL-E V3 do and what hardware is it built on?
> - How does the state machine work and what does it protect against?
> - How does natural language control work end-to-end?

---

## Current status (2026-06-04)

**Hardware:** Physically built. Hardware chapter CLOSED — all future work is simulation-only.

**Gazebo simulation stack: COMPLETE and VALIDATED as of 2026-06-04**
- URDF: cylinder wheels + sphere passive supports (pitch + turn stability), damping, friction, lowered CoM
- Simulated RGBD camera sensor (Gazebo Harmonic rgbd_camera, 5Hz)
- RTAB-Map with subscribe_scan=true (depthimage_to_laserscan → /scan), DetectionRate=1Hz
- Nav2 fully activated: controller, smoother, planner, behavior, bt_navigator, velocity_smoother
- controller_server publishes cmd_vel directly to /cmd_vel (not via broken smoother chain)
- Nav2 costmaps have transform_tolerance=1.5s to handle 1Hz RTAB-Map TF updates
- 4 colored landmark boxes in test_room.sdf (required for visual feature extraction — plain walls have zero features)
- odom_to_tf.py bridges /odom → odom→base_footprint TF
- Requires NVIDIA prime offload: `__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia`
- Workspace root: `~/WALL_E/src` (build/install/log live there)
- **Nav2 autonomous goal validated** — robot drives to goal in sim ✓

**Software checklist:**
- [x] `mega_node` — differential drive math bug in `cmd_callback`
- [x] `mega_node` — mutex missing on publishers called from background threads
- [x] `mega_node` — destructor uses `detach()` instead of `join()` + proper sleep
- [x] URDF — TF chain complete, RealSense D435 optical frames done
- [x] SLAM — RTAB-Map configured for D435 + scan fusion
- [x] Nav2 — Jazzy-compatible params, all nodes active in sim
- [x] `yolo_nav_node` — wired to NavigateToPose action client
- [x] `llama_interpreter_node` — wired to NavigateToPose action client
- [x] **Nav2 autonomous goal validated in sim** ✓
- [ ] **Isaac Sim port** ← next (for RL training + perception coursework)
- [ ] Safety architecture — sensor dropout, localization failure, serial timeout
- [ ] Evaluation — mapping accuracy, nav success rate, fault recovery

**Summer goal:** WALL-E autonomous in Isaac Sim for RL training + perception coursework. Demo video by end of August 2026.

---

## What it is

WALL-E V3 is a tracked mobile robot platform with autonomous navigation, LLM-based natural language control, and visual object navigation. It runs a full ROS2 stack on a Jetson, communicates with three separate microcontrollers over serial/Bluetooth, and supports switching between joystick control and autonomous Nav2 navigation via voice-style text commands parsed by LLaMA 3.2.

---

## Hardware

```
Jetson Orin (ROS2 stack)
├── /dev/rfcomm0  (Bluetooth, 9600 baud)  ← Parallax Propeller (PS2 controller receiver)
├── /dev/ttyUSB0  (Serial, 115200 baud)   ← Arduino Mega (motor driver + wheel encoders)
└── /dev/ttyUSB1  (Serial, 115200 baud)   ← Arduino Uno (MPU-6050 IMU)

Physical:
├── L298N motor driver (differential drive, tank tracks)
├── AS5600 magnetic encoders (wheel odometry)
├── MPU-6050 IMU (pitch/roll for platform stabilization)
├── RealSense D435 (RGB + aligned depth for YOLO nav)
└── LiDAR (for RTAB-Map and Nav2 costmaps)
```

---

## ROS2 node architecture

```
                    [PS2 Controller]
                          │ Bluetooth
               ┌──────────▼──────────┐
               │   controller_node   │ ← /dev/rfcomm0
               │   (C++ serial BT)   │
               └──┬──────┬───────────┘
                  │      │
           /joy_cmd_vel  /state_toggle
           /emergency_stop
                  │      │
               ┌──▼──────▼──────────┐
               │   state_machine    │ ← MANUAL / AUTONOMOUS / IDLE
               └────────┬───────────┘
                        │ /cmd_vel
               ┌────────▼───────────┐
               │    mega_node       │ ← /dev/ttyUSB0  (Arduino Mega)
               │  (C++ serial drv)  │
               └────────────────────┘
                    publishes: /odom, TF odom→base_footprint

               ┌────────────────────┐
               │    uno_bridge      │ ← /dev/ttyUSB1 (Arduino Uno IMU)
               └──┬─────────────────┘
                  │ /wall_e/pitch, /wall_e/roll

               ┌──▼─────────────────┐
               │ llama_interpreter  │ ← ollama LLaMA 3.2:3b
               │   (Python)        │
               └──┬─────────────────┘
                  │ /nav_command, /yolo_target, /state_toggle_cmd

               ┌──▼─────────────────┐
               │  yolo_nav_node     │ ← YOLOv8n + RealSense D435
               │   (Python)        │
               └──┬─────────────────┘
                  │ /goal_pose → Nav2 BT Navigator

               [Nav2 Stack]
               └── publishes /nav_cmd_vel → state_machine
```

---

## Node breakdown

### `controller_node.cpp` — Parallax Bluetooth driver
Reads 8-byte packets from PS2 controller over Bluetooth:

```
Packet (8 bytes): [drive_left, drive_right, euler_x(2B), euler_y(2B), euler_z(2B), changeFlag]
  drive_left/right: 0–255, 127 = stop
  changeFlag: 0=nothing, 1=toggle MANUAL↔AUTONOMOUS, 2=emergency stop
```

Decoding:
```cpp
int steering = -((int)buf[0] - 127);  // negate: 127 center → 0
int throttle = -((int)buf[1] - 127);
twist.linear.x  = (throttle / 124.0f) * 0.5f;   // max 0.5 m/s
twist.angular.z = (steering / 124.0f) * 1.0f;   // max 1.0 rad/s
```

Also sends telemetry **back** to Parallax LCD: pitch angle + current state (MANUAL/AUTONOMOUS/IDLE).
Edge-detection on `changeFlag` (`prev_toggle_`, `prev_estop_`) prevents repeated triggers from held buttons.

### `state_machine.cpp` — Mode router
Three states: `MANUAL`, `AUTONOMOUS`, `IDLE`

```
MANUAL:     /joy_cmd_vel → /cmd_vel   (joystick drives robot)
AUTONOMOUS: /nav_cmd_vel → /cmd_vel   (Nav2 drives robot)
IDLE:       zeros → /cmd_vel + re-publishes zero every 1Hz
```

Transitions:
- `toggle` message: MANUAL ↔ AUTONOMOUS
- `emergency_stop`: any state → IDLE (publishes zero Twist immediately)
- Toggle from IDLE → always back to MANUAL (safe recovery)

State published on `/wall_e/state` at 1Hz → feeds LCD display and LLaMA context.

### `mega_node.cpp` — Arduino Mega serial driver
Sends 8-byte motor commands, receives 20-byte odometry packets:

```
Odometry packet (20 bytes): [0xAA, 0xBB, x(4), y(4), theta(4), lvel(4), rvel(4)]
Sync bytes 0xAA 0xBB prevent misalignment on reconnect.
```

Computes differential drive kinematics:
```cpp
float linear  = (rvel + lvel) / 2.0f;
float angular = (rvel - lvel) / TRACK_WIDTH;  // TRACK_WIDTH = 0.32m
```

Publishes `/odom` + broadcasts TF `odom → base_footprint` from the same data.
Watchdog: sends stop packet if no `/cmd_vel` for 500ms.

### `uno_bridge.cpp` — IMU reader
Reads ASCII lines from Uno: `"P:12.3,R:-1.4\n"` → parses pitch/roll → publishes `/wall_e/pitch`, `/wall_e/roll`.
Used by `controller_node` to show pitch on the Parallax LCD and by LLaMA for status queries.

### `llama_interpreter_node.py` — Natural language control
Runs LLaMA 3.2:3b locally via `ollama`. Input loop reads text commands, builds robot context, sends to LLM, parses JSON response.

**Context injected before every command:**
```python
f"Position: x={x:.2f}m, y={y:.2f}m\nMode: {mode}\nPitch: {pitch:.1f}°\nRoll: {roll:.1f}°\nVelocity: linear={vl:.2f}, angular={va:.2f}"
```

**Response actions:**
| Action | What happens |
|--------|-------------|
| `navigate` | Publishes string command to `/nav_command` |
| `find` | Publishes target class to `/yolo_target` |
| `mode` | Publishes toggle command |
| `estop` | Publishes emergency stop |
| `status` | Prints LLM response to terminal |

System prompt enforces JSON-only output — no markdown, no explanation.

### `yolo_nav_node.py` — Visual object navigation
YOLOv8n detects objects in RGB frame. For the target class, projects 2D detection to 3D using depth + camera intrinsics:

```python
x_cam = (cx - self.cx_) * depth / self.fx_   # pinhole projection
y_cam = (cy - self.cy_) * depth / self.fy_
z_cam = depth

# Convert camera frame → map frame via TF
goal_map = self.tf_buffer_.transform(goal_cam, 'map', timeout=0.1s)
goal_map.pose.position.x -= 0.5   # stop 0.5m in front of object
self.goal_pub_.publish(goal_map)   # → Nav2 goal
```

Camera frame remapping: `camera_color_optical_frame` has Z forward, X right, Y down. The node remaps to ROS convention before publishing the goal.

---

## Serial packet protocols

| Bus | Device | Port | Baud | Packet |
|-----|--------|------|------|--------|
| Bluetooth | Parallax Propeller | /dev/rfcomm0 | 9600 | 8 bytes bidirectional |
| Serial | Arduino Mega | /dev/ttyUSB0 | 115200 | 8 bytes CMD, 20 bytes ODOM |
| Serial | Arduino Uno | /dev/ttyUSB1 | 115200 | ASCII "P:x,R:y\n" |

---

## Key design decisions

**Why state machine instead of just switching Nav2 on/off?**
> Single point of control for `/cmd_vel`. Manual joystick and Nav2 both want to command the robot — without the state machine, they'd fight. The machine also handles IDLE (e-stop) as a safe third state that neither joystick nor Nav2 can override.

**Why LLaMA locally (Ollama) instead of cloud API?**
> Robot is mobile, no guaranteed internet. LLaMA 3.2:3b runs on Jetson Orin without needing connectivity. 3b is fast enough for command parsing — we don't need a large model for JSON extraction from simple sentences.

**Why ASCII serial from Uno instead of binary?**
> Arduino Uno outputs `Serial.print()` which is naturally ASCII. Parsing "P:12.3,R:-1.4\n" is robust to partial reads (line-buffered). The Mega uses binary for higher throughput (odometry at 50Hz would be slow ASCII).

---

## Gotchas

- **Bluetooth latency:** `/dev/rfcomm0` at 9600 baud has ~10ms latency — fine for joystick but too slow for any hard real-time control
- **Edge detection on changeFlag:** Without `prev_toggle_` tracking, holding the toggle button floods the state machine with toggle events and oscillates MANUAL↔AUTONOMOUS at 50Hz
- **TF timeout in YOLO node:** `tf_buffer_.transform()` with `timeout=0.1s` — if TF is stale (Nav2 not running), throws exception. Wrapped in try/except to avoid crash
- **Depth range filter:** `depth < 0.3 or depth > 5.0` filters RealSense noise. D435 minimum reliable depth is ~30cm — values below that are jitter
- **Odometry sync bytes:** Without `0xAA 0xBB` header framing, a single corrupt byte shifts all subsequent packets by 1 — entire odometry becomes garbage. Sync header forces re-alignment

---

## Links

- Related: [[ROS2 Node Design Patterns]], [[Nav2 Stack]], [[Serial Packet Protocols]], [[Sensor Fusion]], [[LLM Tool Calling]], [[YOLOv8 Detection]], [[SLAM & RTAB-Map]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What are the three states of WALL-E's state machine and what routes to /cmd_vel in each? ? MANUAL → /joy_cmd_vel passthrough. AUTONOMOUS → /nav_cmd_vel from Nav2. IDLE → zero Twist always. Emergency stop forces IDLE from any state. Toggle switches MANUAL↔AUTONOMOUS; toggle from IDLE always returns to MANUAL.

How does YOLO object navigation get from pixel detection to a Nav2 goal? ? Pinhole projection: x_cam = (px - cx)*depth/fx, y_cam = (py-cy)*depth/fy, z_cam=depth. Then tf_buffer_.transform() converts camera_color_optical_frame → map frame. Goal is published to /goal_pose 0.5m in front of the object.

What are the three serial buses on WALL-E and what does each carry? ? rfcomm0 (Bluetooth 9600): PS2 controller 8-byte packets bidirectional with Parallax. ttyUSB0 (115200): Arduino Mega — 8-byte motor commands, 20-byte odometry (0xAA 0xBB + x,y,θ,lvel,rvel). ttyUSB1 (115200): Arduino Uno — ASCII "P:x.x,R:x.x\n" IMU data.
