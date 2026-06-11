---

## type: project domain: Simulation & Tooling status: drafted last-reviewed: tags: [px4, mavsdk, drone, aruco, spiral, ros2, python]

# PX4 ArUco Spiral Project

> [!question] Explain it cold
>
> - What does this project do?
> - How does the spiral trajectory work mathematically?
> - How does ArUco precision landing work?

---

## What it is

A ROS2 + MAVSDK Python project with three autonomous drone behaviours running on PX4 SITL:
1. **Spiral search** — fly a parametric spiral trajectory to cover an area
2. **ArUco landing** — detect an ArUco marker and precision-land on it
3. **Waypoint navigation** — fly to a sequence of GPS waypoints

All three use MAVSDK Python async API connected to PX4 SITL on `udp://:14540`.

---

## `spiral_node.py` — Parametric spiral

**Trajectory math:**
```python
radius = 10.0          # metres
num_rotations = 2.0    # full circles
forward_velocity = 3.0 # m/s

# Parametric equations (t: 0 → 1)
x = radius * sin(2π * num_rotations * t)     # lateral oscillation
y = forward_velocity * t                       # forward progress (linear)
z = radius * cos(2π * num_rotations * t)     # vertical oscillation

# This traces a helix projected onto 3D space:
# - x,z oscillate → circular cross-section
# - y advances linearly → forward flight
```

**Execution:**
```python
await drone.action.arm()
await drone.action.takeoff()

async for _ in drone.telemetry.position_velocity_ned():
    await drone.action.goto_location(x, y, absolute_altitude + z, 0)
    await asyncio.sleep(period / 100.0)   # 100 Hz command loop
    t += period / 100.0
    if t >= 1.0: break
```

Note: `goto_location` takes absolute MSL altitude — `absolute_altitude + z` correctly offsets from home altitude.

---

## `aruco_node.py` — Precision landing

```python
await drone.action.goto_location(47.397606, 8.543060, flying_alt, 0)
await drone.camera.start_video_streaming()

async for detection in drone.camera.video_stream():
    if detection.aruco_detection:
        marker_id = detection.aruco_detection.id
        await drone.action.land_on_aruco(marker_id, 0.2)  # 0.2m offset
        break

await drone.action.land()
```

PX4's built-in `land_on_aruco` handles the precision descent — the node just needs to detect the marker ID and hand off to PX4. The `0.2` is the final descent offset in metres.

---

## `waypoint_node.py`

Flies a sequence of GPS waypoints uploaded to PX4 mission. Uses MAVSDK `drone.mission` API to build and upload a `MissionPlan`, then executes it.

---

## MAVSDK async + ROS2 pattern

Mixing MAVSDK's asyncio with ROS2's spin model is tricky:

```python
class SpiralDrone(Node):
    def __init__(self):
        super().__init__('spiral_drone')
        asyncio.ensure_future(self.run())  # schedule async coroutine

    async def run(self):
        drone = System()
        await drone.connect(system_address="udp://:14540")
        ...
```

`asyncio.ensure_future()` schedules the coroutine on the event loop. The ROS2 spin and the asyncio event loop run in the same thread — this works but blocks ROS2 callbacks during `await` calls. For production, use separate threads with `asyncio.run_coroutine_threadsafe`.

---

## PX4 SITL setup (what I ran)

```bash
# Terminal 1 — start PX4 SITL with Gazebo
cd PX4-Autopilot
make px4_sitl gz_x500

# Terminal 2 — start QGroundControl (auto-connects on 14550)
./QGroundControl.AppImage

# Terminal 3 — run the ROS2 node
source install/setup.bash
ros2 run drone_control spiral_node  # or aruco_node, waypoint_node
```

**Health checks before arming:**
```python
async for health in drone.telemetry.health():
    if health.is_global_position_ok and health.is_home_position_ok:
        break  # safe to arm
```
Always wait for GPS fix before arming — PX4 will reject arm if GPS is not healthy.

---

## Key design decisions

**Why MAVSDK Python over mavros/ROS2-native PX4?**
> MAVSDK Python async API is much simpler for behaviour scripting — `await drone.action.arm()` is one line. mavros requires publishing to specific topics and waiting for state feedback. For autonomous behaviour scripts (not real-time control), MAVSDK is the right tool.

**Why `goto_location` in the spiral loop instead of velocity setpoints?**
> `goto_location` lets PX4 handle local path execution — cleaner than computing and streaming velocity setpoints at 100Hz. Trades off precise trajectory tracking for simplicity.

---

## Gotchas

- **Absolute vs relative altitude:** `goto_location()` takes absolute MSL altitude. Forgetting to add `absolute_altitude` sends drone underground. Always fetch home altitude first via `drone.telemetry.home()`
- **asyncio + ROS2 spin:** Both want to own the event loop. `asyncio.ensure_future()` works but MAVSDK awaits block ROS2 callbacks. For actual callback-heavy nodes, run asyncio in a separate thread
- **ArUco node MAVSDK dependency:** `drone.camera.video_stream()` async generator requires PX4 camera MAVLink integration — doesn't work out of the box in basic SITL without camera plugin
- **Pre-arm health check:** If you skip the health check loop and arm immediately, PX4 rejects arm with "global position not ok" — no error from MAVSDK, just silently fails

---

## Links

- Related: [[PX4 SITL & Drone Stack]], [[ROS2 Node Design Patterns]], [[Simulation Environments Comparison]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What are the parametric equations for the spiral trajectory and what shape do they trace? ? x = R·sin(2π·n·t), y = v·t, z = R·cos(2π·n·t). x and z oscillate (circle cross-section), y advances linearly (forward flight). Together they trace a helix — circular in the x-z plane while progressing forward in y.

What is the MAVSDK precision landing flow for ArUco? ? 1) Fly to position above landing area. 2) Start camera video stream. 3) Async-iterate video stream, check for aruco_detection. 4) On detection, call land_on_aruco(marker_id, offset). PX4 handles the precision descent — node just provides the marker ID.

What is the gotcha with goto_location() altitude? ? It takes absolute MSL altitude, not relative-to-home. Must fetch home altitude first via drone.telemetry.home() and add z offset: goto_location(lat, lon, absolute_altitude + z, 0). Forgetting this sends the drone to ground level (or underground).
