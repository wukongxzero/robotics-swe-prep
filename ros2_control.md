---
type: concept
domain: ROS2 & middleware
status: drafted
last-reviewed: 2026-06-02
tags: [ros2, hardware, ros2_control, hardware_interface, controllers]
---

# ros2_control


---

## Core idea

ros2_control is the standard ROS2 hardware abstraction layer. It separates **what** you want to control (joint positions, velocities, efforts) from **how** you communicate with the hardware (serial, CAN, EtherCAT). Controllers write to a shared memory interface; hardware plugins read from it and talk to the actual device. Real-time safe by design.

---

## Three layers

```
┌─────────────────────────────┐
│      Controller Manager     │  ← lifecycle, loads/switches controllers
├─────────────────────────────┤
│         Controllers         │  ← diff_drive_controller, joint_trajectory_controller
│   (read state, write cmd)   │
├─────────────────────────────┤
│    Hardware Interface       │  ← your custom plugin: talks to serial/CAN/etc
│   (state + command buffers) │
└─────────────────────────────┘
```

---

## Key concepts

- **State interface** — what the hardware reports: joint position, velocity, effort
- **Command interface** — what controllers write: velocity command, position setpoint
- **Controller** — reads state, computes output, writes command (e.g. `diff_drive_controller`)
- **Hardware interface plugin** — your code: reads command buffer → sends to hardware, reads hardware → writes state buffer

---

## diff_drive_controller — most relevant for WALL-E

```yaml
# config/diff_drive.yaml
controller_manager:
  ros__parameters:
    update_rate: 50  # Hz

diff_drive_controller:
  ros__parameters:
    left_wheel_names: ["left_track_joint"]
    right_wheel_names: ["right_track_joint"]
    wheel_separation: 0.32
    wheel_radius: 0.08
    publish_odom: true
    odom_frame_id: odom
    base_frame_id: base_footprint
    cmd_vel_topic: /cmd_vel
```

This replaces the manual cmd_vel → serial → Arduino → odom publish chain in mega_node. The controller handles the math; your hardware plugin just moves bytes.

---

## When to use ros2_control vs custom serial node

| Use ros2_control | Use custom serial node (like mega_node) |
|-----------------|----------------------------------------|
| Multiple joint controllers | Single purpose driver |
| Need to switch controllers at runtime | Fixed behavior |
| MoveIt integration needed | Simple differential drive |
| Team knows the pattern | Quick prototype |
| Complex state machines per joint | |

**For WALL-E:** mega_node works fine for the current scope. ros2_control becomes relevant when adding arm joints (FENCE-BOT) or wanting MoveIt integration.

---

## Hardware interface plugin skeleton (C++)

```cpp
#include <hardware_interface/system_interface.hpp>

class WallEHardware : public hardware_interface::SystemInterface {
public:
    hardware_interface::CallbackReturn on_init(
        const hardware_interface::HardwareInfo & info) override {
        // open serial port, configure
        return hardware_interface::CallbackReturn::SUCCESS;
    }

    std::vector<hardware_interface::StateInterface> export_state_interfaces() override {
        // expose: joint velocity, position
        return state_interfaces_;
    }

    std::vector<hardware_interface::CommandInterface> export_command_interfaces() override {
        // expose: joint velocity command
        return command_interfaces_;
    }

    hardware_interface::return_type read(
        const rclcpp::Time &, const rclcpp::Duration &) override {
        // read encoder data from Arduino, write to state buffers
        return hardware_interface::return_type::OK;
    }

    hardware_interface::return_type write(
        const rclcpp::Time &, const rclcpp::Duration &) override {
        // read cmd from command buffers, send to Arduino
        return hardware_interface::return_type::OK;
    }
};
```

---

## Links
- Related: [[ROS2 Node Design Patterns]], [[WALL-E V3]], [[Serial Packet Protocols]], [[Motor Control]]
- Parent: [[00 Knowledge Map]]

---

