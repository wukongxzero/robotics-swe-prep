---
type: concept
domain: ROS2 & middleware
status: drafted
last-reviewed: 2026-06-02
tags: [ros2, testing, gtest, launch_testing, ci]
---

# ROS2 Testing


---

## Core idea

Three levels of testing in ROS2: unit tests (pure logic, no ROS), integration tests (nodes communicating), and launch tests (full system bringup). Most teams only do unit tests — launch tests are the differentiator that catch integration bugs CI misses.

---

## Level 1 — Unit tests (gtest)

Test pure logic without spinning any nodes. Fastest, no ROS2 required.

```cpp
// test/test_drive_math.cpp
#include <gtest/gtest.h>
#include "wall_e_bringup/drive_math.hpp"

TEST(DriveMath, DifferentialFromTwist) {
    auto [left, right] = twist_to_wheel_speeds(0.3, 0.0, 0.32);
    EXPECT_NEAR(left, 0.3, 0.001);
    EXPECT_NEAR(right, 0.3, 0.001);
}

TEST(DriveMath, TurnInPlace) {
    auto [left, right] = twist_to_wheel_speeds(0.0, 1.0, 0.32);
    EXPECT_NEAR(left, -right, 0.001);  // symmetric turn
}
```

```cmake
# CMakeLists.txt
find_package(ament_cmake_gtest REQUIRED)
ament_add_gtest(test_drive_math test/test_drive_math.cpp)
target_link_libraries(test_drive_math drive_math_lib)
```

---

## Level 2 — Integration tests (rclcpp, pytest)

Spin real nodes, publish/subscribe, verify behavior.

```python
# test/test_state_machine.py
import pytest
import rclpy
from rclpy.node import Node
from std_msgs.msg import Bool, String

@pytest.fixture
def node():
    rclpy.init()
    n = Node('test_node')
    yield n
    n.destroy_node()
    rclpy.shutdown()

def test_state_toggle(node):
    received = []
    sub = node.create_subscription(String, '/wall_e/state',
        lambda msg: received.append(msg.data), 10)
    pub = node.create_publisher(Bool, '/state_toggle', 10)

    pub.publish(Bool(data=True))
    rclpy.spin_once(node, timeout_sec=1.0)

    assert 'AUTONOMOUS' in received
```

---

## Level 3 — Launch tests

Spin up the full launch file, verify the system comes up correctly.

```python
# test/test_bringup.launch.py
import unittest
import launch
import launch_ros
import launch_testing
import pytest

@pytest.mark.launch_test
def generate_test_description():
    return launch.LaunchDescription([
        launch_ros.actions.Node(
            package='wall_e_bringup',
            executable='state_machine',
            name='state_machine',
        ),
        launch_testing.actions.ReadyToTest(),
    ])

class TestStateMachineActive(unittest.TestCase):
    def test_node_running(self, proc_output):
        proc_output.assertWaitFor(
            'State machine ready', timeout=10)
```

---

## CMakeLists.txt test setup

```cmake
if(BUILD_TESTING)
  find_package(ament_cmake_gtest REQUIRED)
  find_package(ament_cmake_pytest REQUIRED)
  find_package(launch_testing_ament_cmake REQUIRED)

  # Unit tests
  ament_add_gtest(test_drive_math test/test_drive_math.cpp)

  # Python integration tests
  ament_add_pytest_test(test_state_machine test/test_state_machine.py)

  # Launch tests
  add_launch_test(test/test_bringup.launch.py)
endif()
```

---

## Running tests

```bash
# Build with tests
colcon build --packages-select wall_e_bringup

# Run all tests
colcon test --packages-select wall_e_bringup

# See results
colcon test-result --all --verbose

# Run specific test
colcon test --packages-select wall_e_bringup --pytest-args -k test_state_machine
```

---

## What each level catches

| Level | Catches |
|-------|---------|
| Unit test | Logic bugs in drive math, packet parsing, state transitions |
| Integration test | Topic/service wiring, QoS mismatches, message field errors |
| Launch test | Node startup failures, missing dependencies, TF tree broken at launch |

---

## Links
- Related: [[CI_CD for Robotics]], [[ROS2 Node Design Patterns]], [[Git & Build Systems]]
- Parent: [[00 Knowledge Map]]

---

