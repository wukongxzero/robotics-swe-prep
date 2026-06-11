---

## type: concept domain: SWE & Math Foundations status: drafted last-reviewed: tags: [cpp, python, matlab, programming, swe]

# Programming Languages for Robotics

> [!question] Explain it cold
>
> - When do you write a ROS2 node in C++ vs Python?
> - What is Python used for in a robotics stack that C++ is not?
> - When does MATLAB show up in a robotics engineering role?

---

## Core idea

Three languages cover the full robotics stack:
- **C++** — performance-critical, real-time, hardware-facing code
- **Python** — rapid prototyping, ML/perception, scripting, training pipelines
- **MATLAB** — control system design, simulation, signal processing, academic research

Pick the right one for the layer of the stack you're working in. Most production robotics systems use C++ for the control path and Python for everything else.

---

## C++ — when and why

**Use C++ for:**
| Task | Why C++ |
|------|---------|
| Real-time control loops | Deterministic execution, no GC pauses, cache-friendly |
| ROS2 nodes on the control path | rclcpp, zero-copy with iceoryx, real-time executors |
| Hardware drivers | Direct memory access, register manipulation, low-level I/O |
| SLAM / perception algorithms | Speed — OpenCV, PCL, Eigen are all C++ native |
| Safety-critical code | Predictable memory, RAII, no runtime surprises |
| Isaac Lab custom extensions | C++ plugins for PhysX, custom sensors |
| Embedded (AVR, STM32) | Arduino/AVR-GCC subset, register-level control |

**Key C++ patterns for robotics:**
```cpp
// RAII for hardware resources — always
class SerialPort {
    int fd_;
public:
    explicit SerialPort(const std::string& dev) : fd_(open(dev.c_str(), O_RDWR)) {}
    ~SerialPort() { if (fd_ >= 0) close(fd_); }
    // no copy, move only
    SerialPort(const SerialPort&) = delete;
};

// Real-time safe: no heap alloc in control loop
// BAD in RT loop:
auto msg = std::make_shared<geometry_msgs::msg::Twist>(); // heap alloc

// GOOD in RT loop:
geometry_msgs::msg::Twist msg;  // stack alloc, pre-allocated publisher

// Eigen for math (no dynamic alloc with fixed-size types)
Eigen::Matrix<double, 6, 6> J;   // fixed size → stack, no alloc
Eigen::MatrixXd J_dyn(6, 6);     // dynamic → heap alloc, avoid in RT
```

**C++ robotics libraries:**
- `Eigen` — linear algebra (matrices, vectors, transforms)
- `rclcpp` — ROS2 C++ client
- `OpenCV` — computer vision
- `PCL` — point cloud processing
- `OMPL` — motion planning
- `Pinocchio` — rigid body dynamics

---

## Python — when and why

**Use Python for:**
| Task | Why Python |
|------|-----------|
| ROS2 nodes (non-RT) | rclpy, faster iteration, great for perception nodes |
| ML / perception pipeline | PyTorch, TensorFlow, scikit-learn ecosystem |
| Isaac Lab training scripts | RL training loop, env config, logging |
| Data analysis & plotting | NumPy, Pandas, Matplotlib |
| ROS2 launch files | Python launch API (more powerful than XML) |
| Scripting & automation | Build scripts, CI, data pipeline glue |
| Rapid prototyping | Test an algorithm before porting to C++ |
| URDF/MJCF generation | Programmatic robot model generation |

**Key Python patterns for robotics:**
```python
# ROS2 node pattern
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist

class ControllerNode(Node):
    def __init__(self):
        super().__init__('controller')
        self.pub = self.create_publisher(Twist, '/cmd_vel', 10)
        self.timer = self.create_timer(0.01, self.control_loop)  # 100Hz

    def control_loop(self):
        msg = Twist()
        msg.linear.x = 0.5
        self.pub.publish(msg)

# Isaac Lab env config (Python dataclass pattern)
from isaaclab.envs import ManagerBasedRLEnvCfg
from dataclasses import dataclass

@dataclass
class MyRobotEnvCfg(ManagerBasedRLEnvCfg):
    num_envs: int = 4096
    episode_length_s: float = 10.0
```

**Python robotics libraries:**
- `rclpy` — ROS2 Python client
- `numpy` — arrays, math
- `torch` — ML / RL training
- `isaaclab` — RL environment framework
- `trimesh` — mesh processing, inertia computation
- `pin` (Pinocchio Python) — dynamics
- `scipy` — optimization, signal processing
- `cv2` (OpenCV) — vision

---

## MATLAB — when and why

**Use MATLAB for:**
| Task | Why MATLAB |
|------|-----------|
| Control system design | Control System Toolbox, bode/root locus/step response |
| Simulink modeling | Block-diagram sim for plant + controller before coding |
| Signal processing | DSP Toolbox, FFT, filter design |
| State-space / LQR / Kalman | `lqr()`, `kalman()`, `place()` built-in |
| Academic research / papers | Standard in controls and robotics papers |
| FEA post-processing | Import SolidWorks/ANSYS results, plot stress fields |
| SAE BAJA analysis | Suspension kinematics, load case simulation |
| Robotics Toolbox (Peter Corke) | Forward/inverse kinematics, Jacobian, trajectory gen |

**Key MATLAB patterns:**
```matlab
% LQR design
A = [...]; B = [...];
Q = diag([10, 10, 1, 1]);  % state cost
R = 0.1 * eye(2);           % input cost
K = lqr(A, B, Q, R);        % optimal gain

% Kalman filter
sys = ss(A, B, C, D, dt);
[kest, L, P] = kalman(sys, Qn, Rn);

% Bode plot for loop shaping
C = pid(Kp, Ki, Kd);
G = tf([1], [1 2 1]);
bode(C * G)
margin(C * G)  % gain/phase margins
```

**MATLAB in industry:**
- Common at aerospace, automotive, medical device companies
- Simulink used for model-based design (MBD) — generate C code from blocks
- Less common in pure software robotics startups (Python/C++ dominate there)
- Still asked about in interviews for control systems roles

---

## Decision guide

```
Writing a ROS2 node?
  ├── On the control path (< 1ms deadline, real-time)?  → C++
  └── Perception, planning, UI, logging?                → Python

Writing a training script for RL?          → Python (Isaac Lab)
Writing a hardware driver?                 → C++
Designing a controller (PID, LQR, MPC)?
  ├── Prototyping / tuning?                → MATLAB
  └── Deploying on robot?                  → C++
Doing math / linear algebra in code?
  ├── Performance-critical?               → Eigen (C++)
  └── Research / plotting?               → NumPy (Python) or MATLAB
Writing embedded firmware?                → C++ (Arduino/AVR/STM32 subset)
```

---

## Interview answer: "C++ vs Python for ROS2?"

> "I reach for C++ when I'm on the control path — anything with a hard timing deadline, hardware drivers, or safety-critical logic — because it gives me deterministic execution and real-time safe patterns like RAII and stack allocation. I use Python for everything else: perception nodes, training pipelines, launch files, data analysis. The two coexist in the same ROS2 workspace fine — C++ packages for speed, Python packages for iteration speed. MATLAB comes in when I'm designing or tuning a controller — Control System Toolbox and Simulink let you validate the math before you write a single line of deployable code."

---

## Links

- Related: [[Modern C++ for Robotics]], [[ROS2 Comm Patterns]], [[Isaac Lab]], [[PID Control]], [[LQR]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

When do you write a ROS2 node in C++ vs Python? ? C++ for the control path: hard timing deadlines, hardware drivers, real-time executors, safety-critical code. Python for everything else: perception, planning, training, launch files, logging.

What does MATLAB give you that Python doesn't for control design? ? Control System Toolbox built-ins (lqr, kalman, place, bode, margin), Simulink block-diagram modeling, and industry-standard tools for model-based design that generate deployable C code from Simulink blocks.

What is Eigen used for in C++ robotics code? ? Linear algebra — fixed-size matrices/vectors (Eigen::Matrix<double,6,6>) allocate on the stack with no heap alloc, making them real-time safe. Used for Jacobians, transforms, state vectors everywhere in control and kinematics code.
