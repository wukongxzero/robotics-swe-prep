---

## type: concept domain: ROS2 & Middleware status: drafted last-reviewed: 2026-06-02 tags: [ros2, cpp, node, design, patterns, real-time]

# ROS2 Node Design Patterns

> [!question] Explain it cold
>
> - What is the canonical structure of a well-written ROS2 C++ node?
> - How do you safely share data between a subscriber callback and a timer?
> - When do you need a background thread, and how do you manage its lifetime?

---

## Core idea

A well-written ROS2 node has a clear structure: all setup in the constructor, no logic in `main()` beyond `spin`, RAII for every resource, and explicit thread safety wherever callbacks and timers share data. The goal is a node you can read top to bottom and immediately understand what it subscribes to, what it publishes, and what can go wrong.

---

## Canonical C++ node structure

```cpp
#include <rclcpp/rclcpp.hpp>
#include <some_msgs/msg/something.hpp>
#include <mutex>

class MyNode : public rclcpp::Node {
public:
    MyNode() : Node("my_node") {
        // 1. Declare and read parameters
        this->declare_parameter("rate_hz", 50.0);
        double rate = this->get_parameter("rate_hz").as_double();

        // 2. Create subscriptions
        sub_ = this->create_subscription<some_msgs::msg::Something>(
            "/input_topic", 10,
            std::bind(&MyNode::callback, this, std::placeholders::_1));

        // 3. Create publishers
        pub_ = this->create_publisher<some_msgs::msg::Something>("/output_topic", 10);

        // 4. Create timers (control loop, watchdog, etc.)
        timer_ = this->create_wall_timer(
            std::chrono::duration<double>(1.0 / rate),
            std::bind(&MyNode::controlLoop, this));

        RCLCPP_INFO(this->get_logger(), "Node ready at %.1f Hz", rate);
    }

    ~MyNode() {
        // Clean up non-RAII resources (sockets, serial ports, threads)
    }

private:
    void callback(const some_msgs::msg::Something::SharedPtr msg) {
        std::lock_guard<std::mutex> lock(mutex_);
        latest_ = msg;
        has_data_ = true;
    }

    void controlLoop() {
        std::lock_guard<std::mutex> lock(mutex_);
        if (!has_data_) return;
        // process latest_
    }

    rclcpp::Subscription<some_msgs::msg::Something>::SharedPtr sub_;
    rclcpp::Publisher<some_msgs::msg::Something>::SharedPtr pub_;
    rclcpp::TimerBase::SharedPtr timer_;

    some_msgs::msg::Something::SharedPtr latest_;
    std::mutex mutex_;
    bool has_data_ = false;
};

int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<MyNode>());
    rclcpp::shutdown();
}
```

---

## Pattern 1 — Subscriber + Timer with mutex (from FENCE-BOT)

The most common pattern: subscriber receives data, timer consumes it at a fixed rate.

```cpp
// From FENCE-BOT robot_controller.cpp — real production code
class RobotController : public rclcpp::Node {
public:
    RobotController() : Node("robot_controller") {
        auto qos = rclcpp::QoS(10).best_effort();  // match VR publisher QoS
        subscription_ = this->create_subscription<geometry_msgs::msg::PoseStamped>(
            "/vr_pose", qos,
            std::bind(&RobotController::poseCallback, this, std::placeholders::_1));

        // Open UDP socket to Isaac Lab
        sock_ = socket(AF_INET, SOCK_DGRAM, 0);
        isaac_addr_.sin_family = AF_INET;
        isaac_addr_.sin_port   = htons(5006);
        inet_pton(AF_INET, "127.0.0.1", &isaac_addr_.sin_addr);

        // 50Hz control loop — decouples from subscription rate
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(20),
            std::bind(&RobotController::controlLoop, this));
    }

    ~RobotController() { close(sock_); }  // RAII for socket

private:
    void poseCallback(const geometry_msgs::msg::PoseStamped::SharedPtr msg) {
        std::lock_guard<std::mutex> lock(mutex_);
        latest_pose_ = msg;
        new_data_ = true;
    }

    void controlLoop() {
        std::lock_guard<std::mutex> lock(mutex_);
        if (!latest_pose_ || !new_data_) return;
        new_data_ = false;
        // pack and send UDP to Isaac Lab
        Packet pkt{ ... };
        sendto(sock_, &pkt, sizeof(pkt), 0, ...);
        RCLCPP_INFO_THROTTLE(this->get_logger(), *this->get_clock(), 1000,
            "Forwarding pose");  // throttled log = 1 msg/sec not 50/sec
    }

    rclcpp::Subscription<...>::SharedPtr subscription_;
    rclcpp::TimerBase::SharedPtr timer_;
    geometry_msgs::msg::PoseStamped::SharedPtr latest_pose_;
    std::mutex mutex_;
    bool new_data_ = false;
    int sock_;
    sockaddr_in isaac_addr_;
};
```

**Key decisions here:**
- `best_effort` QoS matches the VR publisher — if QoS mismatches, subscription silently receives nothing
- Timer at 50Hz decouples control loop rate from message arrival rate
- `new_data_` flag prevents sending stale data on every tick
- `RCLCPP_INFO_THROTTLE` prevents log spam at 50Hz

---

## Pattern 2 — Background read thread (from WALL-E)

When you need to read from hardware (serial, socket) in a blocking loop alongside ROS2 spinning.

```cpp
// From WALL-E mega_node.cpp — serial hardware driver node
class MegaNode : public rclcpp::Node {
public:
    MegaNode() : Node("mega_node") {
        fd_ = open("/dev/ttyUSB0", O_RDWR | O_NOCTTY);
        configure_serial(fd_);

        cmd_sub_  = create_subscription<Twist>("/cmd_vel", 10,
            std::bind(&MegaNode::cmd_callback, this, _1));
        odom_pub_ = create_publisher<Odometry>("/odom", 10);
        tf_broadcaster_ = std::make_shared<tf2_ros::TransformBroadcaster>(this);

        // Watchdog: stop motors if no cmd_vel for 500ms
        last_cmd_ = now();
        watchdog_ = create_wall_timer(100ms, std::bind(&MegaNode::watchdog, this));

        // Background thread reads serial — never blocks the ROS2 executor
        read_thread_ = std::thread(&MegaNode::read_loop, this);
        read_thread_.detach();
    }

    ~MegaNode() {
        running_ = false;                          // signal thread to stop
        std::this_thread::sleep_for(100ms);        // let it exit
        send_packet(127, 127);                     // stop motors safely
        close(fd_);                                // then close hardware
    }

private:
    void read_loop() {
        while (running_ && rclcpp::ok()) {
            uint8_t c;
            if (read(fd_, &c, 1) > 0) {
                // accumulate into packet, parse on complete frame
                // publishes /odom and TF from inside this thread
            }
        }
    }

    bool running_ = true;
    std::thread read_thread_;
    int fd_;
    rclcpp::Time last_cmd_;
    // ...
};
```

**Key decisions here:**
- `running_` flag signals the thread before destroying — avoids race on `fd_` close
- Destructor order: signal → wait → stop motors → close. Order matters.
- Read thread calls `odom_pub_->publish()` — this is safe because `rclcpp` publishers are thread-safe
- Watchdog timer is a separate ROS2 timer, not in the read thread — clean separation of concerns

---

## Pattern 3 — Watchdog timer

Always add a watchdog to hardware nodes that receive commands:

```cpp
// In constructor:
last_cmd_ = now();
watchdog_ = create_wall_timer(100ms, [this]() {
    if ((now() - last_cmd_).seconds() > 0.5) {
        stop_motors();  // or send zero velocity
    }
});

// In cmd callback:
void cmd_callback(const Twist::SharedPtr msg) {
    last_cmd_ = now();  // reset watchdog
    send_command(msg);
}
```

If the upstream node crashes or the network drops, the robot stops within 500ms instead of continuing last command forever.

---

## Thread safety rules

| Situation | Rule |
|-----------|------|
| Callback + timer share data | `std::mutex` + `std::lock_guard` — always |
| Background thread publishes | Safe — `rclcpp::Publisher` is thread-safe |
| Background thread calls `RCLCPP_INFO` | Safe |
| Background thread modifies node state | Needs mutex |
| Multiple callbacks share data | Use `MutuallyExclusiveCallbackGroup` OR mutex |

---

## QoS — the silent failure

Mismatched QoS = subscriber receives nothing, no error:

```cpp
// Publisher uses reliable:
pub = create_publisher<Msg>("/topic", rclcpp::QoS(10));  // default = reliable

// Subscriber uses best_effort — MISMATCH. Receives nothing.
sub = create_subscription<Msg>("/topic", rclcpp::QoS(10).best_effort(), cb);

// Fix: match QoS on both sides
auto qos = rclcpp::QoS(10).best_effort();
// OR use rclcpp::SensorDataQoS() for sensor topics
```

**Common QoS profiles:**
```cpp
rclcpp::QoS(10)                    // reliable, volatile, depth 10
rclcpp::SensorDataQoS()            // best_effort, volatile — for sensors
rclcpp::SystemDefaultsQoS()        // whatever rmw defaults are
rclcpp::QoS(1).transient_local()   // latched topic — late subscribers get last msg
```

---

## Package structure (best practice)

```
my_robot_pkg/
├── CMakeLists.txt
├── package.xml
├── include/my_robot_pkg/
│   └── my_node.hpp        ← class declaration
├── src/
│   ├── my_node.cpp        ← class implementation
│   └── main.cpp           ← just rclcpp::init + spin
├── launch/
│   └── my_robot.launch.py
├── config/
│   └── params.yaml
└── test/
```

**Launch file pattern:**
```python
from launch import LaunchDescription
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    config = os.path.join(
        get_package_share_directory('my_robot_pkg'), 'config', 'params.yaml')

    return LaunchDescription([
        Node(
            package='my_robot_pkg',
            executable='my_node',
            name='my_node',
            parameters=[config],
            remappings=[('/cmd_vel', '/robot/cmd_vel')],
            output='screen',
        )
    ])
```

---

## Common mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Logic in `main()` | Hard to test, can't compose | Move everything into Node class |
| No mutex on shared data | Race condition — callbacks + timers run concurrently | `lock_guard` on every shared access |
| QoS mismatch | Silent: no data received, no error | Always match QoS explicitly |
| `RCLCPP_INFO` in 50Hz loop | Log spam fills terminal | Use `RCLCPP_INFO_THROTTLE` |
| Raw pointer to hardware | No cleanup on crash | RAII: file descriptor in class, close in destructor |
| Blocking call in callback | Blocks executor, all other callbacks stall | Move blocking work to thread or async timer |
| Forgetting `new_data_` flag | Sends stale data every tick | Track freshness with bool flag |

---

## Links

- Related: [[ROS2 Executors]], [[ROS2 Comm Patterns]], [[ROS2 QoS]], [[Modern C++ for Robotics]], [[FENCE-BOT]], [[WALL-E V3]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What is the canonical structure of a ROS2 C++ node? ? Class inherits rclcpp::Node. Constructor: declare params, create subscriptions/publishers/timers. Callbacks: short, lock mutex, store data. Control loop in timer. RAII for hardware. main() only does init/spin/shutdown.
<!--SR:!2026-05-27,0,230-->

How do you safely share data between a ROS2 subscriber callback and a timer? ? std::mutex + std::lock_guard in both the callback (write) and the timer (read). Also use a bool flag (new_data_) so the timer knows whether data has been updated since the last tick.
<!--SR:!2026-05-30,3,250-->

Why does a QoS mismatch silently drop all messages? ? ROS2 DDS matches subscribers to publishers only if QoS is compatible. Reliable publisher won't match best_effort subscriber. No error is thrown — the subscription just receives nothing. Always explicitly match QoS on both sides.

What pattern do you use when you need to read from serial/hardware blocking in a ROS2 node? ? Background thread with a running_ bool flag. Thread calls read() in a loop. Destructor sets running_=false, waits briefly, then closes the hardware. Publishers called from the thread are safe — rclcpp publishers are thread-safe.

What does a mutex protect — the mutex itself or the data? ? The DATA — latest_pose_, new_data_, etc. The mutex is the lock mechanism, not the thing being protected. "Protecting the mutex" is a common misstatement.

What is RAII? Give a concrete ROS2 example. ? Resource Acquisition Is Initialization — a resource is tied to object lifetime: acquired in constructor, released in destructor. Example: serial port fd_ opened in MegaNode constructor, closed in destructor. No explicit cleanup needed elsewhere.

What is the correct destructor order for a node with a background read thread? ? (1) Set running_=false to signal the thread. (2) Wait for it to exit (join or sleep). (3) Stop hardware safely. (4) Close the file descriptor. Closing fd_ first causes the thread to read() on a closed handle — undefined behavior.

Why does the new_data_ flag exist in the subscriber+timer pattern? ? Prevents sending stale data every tick. Without it, the 50Hz timer acts on the last received message every cycle even if no new message arrived. The flag resets to false after each use so the timer skips ticks with no fresh data.

QoS compatibility direction — which side must be more restrictive? ? The publisher sets the contract; the subscriber must match or be more permissive. A best_effort subscriber won't match a reliable publisher — subscriber receives nothing silently. Fix: match QoS explicitly on both sides.
