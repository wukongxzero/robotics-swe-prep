---

## type: concept domain: Simulation & Tooling status: drafted last-reviewed: tags: [px4, sitl, drone, mavlink, uav, ros2]

# PX4 SITL & Drone Stack

> [!question] Explain it cold
>
> - What is PX4 and what does SITL mean?
> - How does the full PX4 SITL stack connect — firmware, simulator, GCS, and your code?
> - What protocol does everything communicate over?

---

## Core idea

**PX4** is the open-source autopilot firmware that runs on flight controller hardware (Pixhawk, etc.) and handles all low-level flight: attitude estimation, motor mixing, flight modes, failsafes. **SITL** (Software In The Loop) runs the actual PX4 firmware binary on your laptop in simulation — the same code that would run on hardware — connected to a physics simulator (Gazebo, jMAVSim) for sensor feedback.

---

## Full stack architecture

```
┌─────────────────────────────────────────────────────┐
│                  Your code / ROS2                    │
│         (MAVSDK Python / mavros / ros2_mavros)       │
└───────────────────┬─────────────────────────────────┘
                    │ MAVLink (UDP 14540)
┌───────────────────▼─────────────────────────────────┐
│              PX4 SITL (firmware binary)              │
│   Attitude estimator │ Position controller │ Mixer   │
└──────────┬──────────────────────────┬───────────────┘
           │ MAVLink (UDP 14550)      │ Gazebo plugin (sensor/actuator)
           │                          │
┌──────────▼──────────┐   ┌──────────▼───────────────┐
│   QGroundControl    │   │    Gazebo / jMAVSim       │
│  (GCS - monitor,    │   │  (physics + sensors)      │
│   mission planning) │   └──────────────────────────┘
└─────────────────────┘
```

**Port conventions:**
- `14550` → QGroundControl listens here
- `14540` → MAVSDK / offboard code connects here
- `4560` → Gazebo ↔ PX4 sensor/actuator bridge

---

## Key facts

**MAVLink:**
- Lightweight binary protocol for drone communication (telemetry, commands, sensor data)
- Every message has a message ID, system ID, component ID
- Heartbeat at 1Hz keeps connection alive — if it stops, PX4 triggers failsafe
- Key messages: `HEARTBEAT`, `COMMAND_LONG`, `SET_POSITION_TARGET_LOCAL_NED`, `ATTITUDE`, `LOCAL_POSITION_NED`

**Flight modes (the ones that matter for dev):**
| Mode | What it does |
|------|-------------|
| `OFFBOARD` | Your code sends setpoints — full external control |
| `STABILIZED` | Attitude control, pilot inputs |
| `POSITION` | GPS hold, position control |
| `MISSION` | Executes uploaded waypoint mission |
| `HOLD` | Loiters at current position |
| `RTL` | Return to launch |

**OFFBOARD mode rules:**
- Must send setpoints at > 2Hz before entering offboard, and continuously while in it
- If setpoints stop → PX4 exits offboard mode and triggers failsafe
- Arm → set mode to OFFBOARD → start sending setpoints (order matters)

---

## SITL setup (what I've done)

```bash
# 1. Clone PX4-Autopilot
git clone https://github.com/PX4/PX4-Autopilot.git --recursive
cd PX4-Autopilot

# 2. Install dependencies
bash ./Tools/setup/ubuntu.sh

# 3. Build and launch SITL with Gazebo
make px4_sitl gz_x500          # x500 quadrotor in Gazebo
# OR with jMAVSim (lighter, no 3D viz)
make px4_sitl jmavsim

# 4. PX4 shell appears — drone is "flying" in sim
# QGroundControl auto-connects on UDP 14550

# 5. Connect your code via MAVSDK (Python)
pip install mavsdk
```

**MAVSDK Python offboard example:**
```python
import asyncio
from mavsdk import System
from mavsdk.offboard import OffboardError, VelocityNedYaw

async def run():
    drone = System()
    await drone.connect(system_address="udp://:14540")

    # Wait for connection
    async for state in drone.core.connection_state():
        if state.is_connected:
            break

    # Arm
    await drone.action.arm()

    # Must send setpoint before entering offboard
    await drone.offboard.set_velocity_ned(VelocityNedYaw(0, 0, 0, 0))
    await drone.offboard.start()

    # Fly
    await drone.offboard.set_velocity_ned(VelocityNedYaw(0, 0, -1.0, 0))  # up 1 m/s
    await asyncio.sleep(3)

    await drone.offboard.stop()
    await drone.action.land()

asyncio.run(run())
```

**With ROS2 (ros2_mavros / XRCE-DDS):**
```bash
# Modern PX4 ↔ ROS2 via uXRCE-DDS (replaces mavros for PX4 v1.14+)
# PX4 publishes to ROS2 topics directly:
#   /fmu/out/vehicle_local_position
#   /fmu/out/vehicle_attitude
#   /fmu/in/offboard_control_mode
#   /fmu/in/trajectory_setpoint
```

---

## Common SITL pain points I've hit

| Problem | Cause | Fix |
|---------|-------|-----|
| Drone won't arm in SITL | Preflight checks failing (GPS, RC) | `param set COM_RCL_EXCEPT 4` to bypass RC loss check in sim |
| Offboard mode rejected | Setpoints not being sent before mode switch | Send at least one setpoint, then switch to OFFBOARD |
| MAVLink connection refused | Wrong port or firewall | Check `udp://:14540` vs `14550`, confirm PX4 is running |
| Gazebo model not spawning | Missing `PX4_SIM_MODEL` env var | `export PX4_SIM_MODEL=gz_x500` |
| SITL resets immediately | Build error or missing dependencies | Run `make clean` then rebuild |
| Drone drifts in position | No GPS fix in sim | Wait for `GPS_RAW_INT` to show fix, or use `make px4_sitl gz_x500` (has GPS) |

---

## PX4 coordinate frames

```
NED (North-East-Down) — PX4 native frame:
  X = North, Y = East, Z = Down (positive Z = deeper underground)
  Yaw = 0 points North, increases clockwise

ENU (East-North-Up) — ROS2 / Gazebo native:
  X = East, Y = North, Z = Up

Conversion NED → ENU: x_enu = y_ned, y_enu = x_ned, z_enu = -z_ned
```

**This mismatch is a common bug** — always be explicit about which frame your setpoints are in.

---

## Links

- Related: [[Simulation Environments Comparison]], [[URDF & CAD Pipeline]], [[ROS2 Comm Patterns]], [[VR Teleop Pipeline]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is PX4 SITL and how does it differ from hardware? ? SITL runs the actual PX4 firmware binary on your laptop connected to a physics simulator (Gazebo/jMAVSim) for sensor feedback. Same code as real hardware — only the sensor/actuator interface is simulated. Lets you test flight logic without a physical drone.

What protocol does everything in the PX4 stack communicate over? ? MAVLink — lightweight binary protocol with message IDs, system IDs, heartbeat at 1Hz. QGroundControl on UDP 14550, MAVSDK/offboard code on UDP 14540, Gazebo bridge on 4560.

What are the rules for OFFBOARD mode in PX4? ? 1) Must send setpoints at >2Hz before entering offboard. 2) Must keep sending continuously — if setpoints stop, PX4 exits offboard and triggers failsafe. 3) Order: arm → start sending setpoints → switch to OFFBOARD mode.

NED vs ENU — what's the difference and why does it matter? ? NED (North-East-Down) is PX4's native frame — Z points down. ENU (East-North-Up) is ROS2/Gazebo native — Z points up. Conversion: x_enu=y_ned, y_enu=x_ned, z_enu=-z_ned. Mixing them up causes inverted altitude commands — a common SITL bug.
