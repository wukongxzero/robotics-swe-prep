# micro-ROS

## What is micro-ROS?

micro-ROS brings the ROS2 API to microcontrollers (MCUs) — things like STM32, ESP32, Arduino, Teensy — that can't run a full Linux + ROS2 stack.

**Key idea:** The MCU runs a stripped-down ROS2 client library (`rclc`) and talks to a full ROS2 network via an **agent** running on a Linux host (Jetson, RPi, PC).

```
[MCU]  <-- UART/USB/WiFi -->  [micro-ROS Agent (Linux)]  <-->  [ROS2 Network]
rclc node                      bridges DDS ↔ MCU transport
```

---

## How it differs from ROS2

| Feature | ROS2 (Linux) | micro-ROS (MCU) |
|---|---|---|
| Transport | DDS (UDP/TCP) | Micro XRCE-DDS over serial/UDP |
| Executor | rclcpp MultiThreaded | rclc single-threaded spin |
| Memory | Dynamic (heap) | Static allocations, no dynamic alloc |
| Middleware | rmw_fastrtps etc. | rmw_microxrcedds |
| RTOS | None (Linux scheduler) | FreeRTOS, Zephyr, NuttX, bare-metal |

---

## Transport Layers (Micro XRCE-DDS)

**XRCE** = eXtremely Resource Constrained Environments.

The MCU speaks a compact binary protocol to the agent, which translates to full DDS on the ROS2 side.

Supported transports:
- **UART/USB serial** — simplest, reliable, low latency
- **UDP over WiFi** — ESP32, wireless robots
- **CAN FD** — automotive/industrial (experimental)

---

## micro-ROS Implementation Patterns

### 1. Publisher node (C, rclc)

```c
#include <rcl/rcl.h>
#include <rclc/rclc.h>
#include <std_msgs/msg/int32.h>

rcl_publisher_t publisher;
std_msgs__msg__Int32 msg;

// Init
rclc_publisher_init_default(&publisher, &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Int32), "my_topic");

// Publish in loop
msg.data = 42;
rcl_publish(&publisher, &msg, NULL);
```

### 2. Subscriber node

```c
rcl_subscription_t subscriber;
std_msgs__msg__Int32 recv_msg;

void callback(const void* msg_in) {
    const std_msgs__msg__Int32* msg = (const std_msgs__msg__Int32*)msg_in;
    // handle msg->data
}

rclc_subscription_init_default(&subscriber, &node,
    ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Int32), "my_topic");

rclc_executor_add_subscription(&executor, &subscriber,
    &recv_msg, &callback, ON_NEW_DATA);
```

### 3. Executor spin (replaces rclcpp::spin)

```c
rclc_executor_t executor;
rclc_executor_init(&executor, &support.context, 1, &allocator);
// add publishers/subscribers/timers...
rclc_executor_spin(&executor);  // blocking
// OR
rclc_executor_spin_some(&executor, RCL_MS_TO_NS(10));  // non-blocking tick
```

**Key constraint:** rclc executor is single-threaded. On FreeRTOS, run it in one task. Don't share across tasks without mutexes.

---

## micro-ROS + Pathfinding (A\*)

A common RSE pattern: MCU runs a local A\* planner on a small occupancy grid (e.g., 20x20 costmap received from the agent), publishes the resulting path back to ROS2.

```
[ROS2 Nav2 costmap] --> agent --> MCU
                                   ↓
                              A* on local grid
                                   ↓
                         MCU --> agent --> [ROS2 path topic]
```

### Why run A\* on the MCU?
- Offloads compute from the main CPU
- Low-latency local replanning (obstacle avoidance in tight loop)
- MCU is physically closer to motor controllers — shorter feedback path

### Practical constraints for A\* on MCU:
| Concern | Mitigation |
|---|---|
| No dynamic alloc | Pre-allocate `g[][]` and priority queue with fixed max size |
| No `std::priority_queue` | Use a fixed-size min-heap or sorted array for small grids |
| Stack depth | Keep recursion out — use iterative A\* |
| Timing | Run planner in a FreeRTOS task with known period |

### Minimal C A\* on MCU (pseudocode pattern)

```c
#define MAX_NODES 400  // 20x20 grid

typedef struct { int f, g, r, c; } Node;
Node heap[MAX_NODES];
int heap_size = 0;
int g_cost[20][20];

// min-heap push/pop manually (no STL on bare-metal)
// same algorithm logic as C++ version — just static memory
```

---

## Timing Constraints

On FreeRTOS with micro-ROS:

```c
// Recommended pattern: dedicate one task to micro-ROS spin
void micro_ros_task(void* arg) {
    // init node, pubs, subs, executor...
    while (1) {
        rclc_executor_spin_some(&executor, RCL_MS_TO_NS(10));
        vTaskDelay(pdMS_TO_TICKS(10));  // yield to other tasks
    }
}
```

- Keep the spin period ≤ your fastest publisher period
- Don't block inside a callback — offload heavy work (like A\*) to a separate FreeRTOS task and use a queue to pass results back

---

## Supported MCUs

| Board | RTOS | Transport |
|---|---|---|
| STM32 (Nucleo, etc.) | FreeRTOS | UART |
| ESP32 | FreeRTOS | UART or UDP/WiFi |
| Teensy 4.x | FreeRTOS | USB serial |
| Raspberry Pi Pico | FreeRTOS / bare-metal | UART |
| Arduino (limited) | bare-metal | UART |

---

## Quick Setup (Linux host side)

```bash
# Run the micro-ROS agent (bridges MCU <-> ROS2)
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0 -b 115200

# Check topics from MCU
ros2 topic list
ros2 topic echo /my_topic
```

---

## Flashcards

What is the role of the micro-ROS agent? ? Runs on the Linux host; bridges the XRCE-DDS protocol from the MCU to the full DDS ROS2 network. Without it, the MCU node is invisible to the ROS2 graph.

What transport does micro-ROS use under the hood? ? Micro XRCE-DDS — a compact binary protocol designed for resource-constrained devices. Runs over UART, USB serial, or UDP/WiFi.

How does the rclc executor differ from rclcpp? ? Single-threaded, no dynamic allocation, uses `spin_some` or `spin` — designed for MCU task loops, not Linux multi-threaded scheduling.

Why avoid dynamic allocation in micro-ROS on bare-metal? ? MCUs have no heap manager and limited RAM. Dynamic alloc risks fragmentation and non-deterministic timing — critical for real-time control.

How do you run A\* on an MCU safely? ? Pre-allocate all arrays (g-cost grid, fixed-size heap), use iterative not recursive A\*, run in a dedicated FreeRTOS task separate from the micro-ROS spin task.

Where does micro-ROS fit in Nav2? ? MCU acts as a low-level replanner or sensor node. Nav2 global planner (A\*/Dijkstra) runs on the main computer; micro-ROS MCU can do local obstacle reactions faster.
