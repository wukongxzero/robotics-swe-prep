---

## type: concept domain: Real-time & embedded status: drafted last-reviewed: tags: [embedded, can, fieldbus, real-time, differentiator]

# CAN Bus

> [!star] Differentiator territory CAN on a production multi-arm surgical system (Articulus) is real industrial/medical fieldbus experience most candidates don't have. Pair it with [[iceoryx Zero-Copy]] and [[Motor Control]] for the full real-time-systems story.

> [!question] Explain it cold
> 
> - What is CAN and why is it used for motor/actuator networks?
> - How does arbitration work, and why is it non-destructive?
> - What makes CAN deterministic for high-priority messages?

---

## Core idea

CAN (Controller Area Network) is a multi-master, message-based serial bus designed for robust real-time communication among distributed nodes (motor controllers, sensors) over a two-wire differential pair. Messages are addressed by **ID = priority**, and bus arbitration is non-destructive, so the highest-priority message always wins without corruption or retransmission.

## Key facts & formulas

- **Differential pair** (CAN_H / CAN_L): noise-immune, terminated with 120 Ω at both ends. Built for electrically noisy environments (motors, surgical OR).
- **Message-based, not address-based**: frames carry an **identifier** that is both the content label and the **priority** (lower ID = higher priority). Nodes filter for IDs they care about.
- **Arbitration (CSMA/CD with non-destructive bitwise arbitration)**: nodes start transmitting simultaneously; the dominant bit (0) wins over recessive (1). A node that sees a dominant bit while sending recessive backs off — _without_ the winning message being corrupted. So the highest-priority frame transmits with zero retransmission delay. This is the determinism property.
- **Frame**: ID + control + up to 8 data bytes (classic CAN) + CRC + ACK. **CAN FD** extends payload to 64 bytes and higher data-phase bitrate.
- **Error handling**: built-in CRC, ACK slot, error frames, and fault confinement (error-active / error-passive / bus-off states) — a misbehaving node removes itself.
- **Bitrate vs length tradeoff**: 1 Mbit/s up to ~40 m; longer buses run slower (propagation must fit the bit time for arbitration to work).


## 120 Ω termination — why and what happens without it

**Root cause:** At high bitrates (up to 1 Mbit/s), CAN wire behaves as a transmission line, not just a conductor. Signals that hit an unterminated end reflect back and corrupt the original signal.

**The fix:** 120 Ω resistors at both ends of the bus match the characteristic impedance of the twisted pair cable. A matched termination absorbs the signal — no reflection.

**Why 120 Ω specifically?** It's the characteristic impedance of standard CAN twisted pair, derived from the physical properties of the wire (inductance and capacitance per meter). Not arbitrary.

**Why two resistors?** One at each physical end of the bus — the bus has two ends, both must be terminated.

```
Node A ──[120Ω]──────────────────────[120Ω]── Node B
                  Node C    Node D
                  (no termination — middle nodes never get resistors)
```

**What happens without termination:**
- Signal reflections cause voltage spikes on CAN_H / CAN_L
- Receivers misread dominant/recessive levels
- CRC errors → error frames → TEC climbs → nodes go error-passive or bus-off
- On `candump`: error frames everywhere, frames randomly missing ACK

**Bitrate vs cable length interaction:** Longer cable = higher propagation delay. At high bitrates the bit time shrinks — if propagation delay exceeds the bit time, arbitration breaks. This is why:
- 1 Mbit/s → max ~40 m
- 125 kbit/s → max ~500 m
- Longer bus = must run slower

**Debug rule:** Bus works at short distances but fails at longer cables or higher bitrates → check termination and bitrate/length tradeoff first.

---

## SocketCAN on Linux (from actual code)

```c
#include <linux/can.h>
#include <linux/can/raw.h>

// 1. Create raw CAN socket
int soc = socket(PF_CAN, SOCK_RAW, CAN_RAW);

// 2. Resolve interface name → index and bind
struct sockaddr_can addr;
strcpy(addr.ifr_name, "can0");
ioctl(soc, SIOCGIFINDEX, &addr);
addr.can_family = AF_CAN;
bind(soc, (struct sockaddr*)&addr, sizeof(addr));

// 3. Build and send frame
struct can_frame frame;
frame.can_id  = 0x1555AAB0;  // 29-bit extended ID (>0x7FF = extended)
frame.can_dlc = 8;
frame.data[0] = 0x01;
write(soc, &frame, sizeof(frame));

close(soc);
```

**Bring up interface:**
```bash
sudo ip link set can0 type can bitrate 1000000
sudo ip link set can0 up
# Virtual CAN for testing (no hardware):
sudo modprobe vcan && sudo ip link add dev vcan0 type vcan && sudo ip link set vcan0 up
candump vcan0          # listen
cansend vcan0 123#DEADBEEF  # send test frame
```

## Motor control command sequence (from your code)

Extended CAN IDs encode device address + command type — common pattern for COTS motor controllers:

```c
send_can_message(soc, 0x1555AAB0, {0x01, 0x01, ...});   // 1. Init driver
send_can_message(soc, 0x1555AAB1, {0x00, 0x00, 0x01}); // 2. Absolute position mode
send_can_message(soc, 0x1555AAB2, {0x00,0x00,0x00, 0x0A, 0x32, 0x96}); // 3. PI gains
send_can_message(soc, 0x1555AAB3, {0x01,0xF4, 0x01,0xF4}); // 4. Accel 500rpm/s (0x1F4=500)
// Loop: set target → execute
send_can_message(soc, 0x1555AAB5, {0x00,0x00,0x00,0x3D, 0x1F,0x40}); // 5a. +20° at 800rpm
send_can_message(soc, 0x1555AAB6, {0x01, ...});          // 5b. Run
send_can_message(soc, 0x1555AAB5, {0x80,0x00,0x00,0x3D, 0x1F,0x40}); // 6a. -20° at 800rpm
send_can_message(soc, 0x1555AAB6, {0x01, ...});          // 6b. Run
```

**Setup order always:** init → mode → gains → accel → position targets

## CAN frame format — bit by bit

Classic CAN frame (data frame):

```
| SOF | ID (11b) | RTR | IDE | r0 | DLC (4b) | Data (0–64B) | CRC (15b) | CRC delim | ACK | ACK delim | EOF (7b) | IFS |
```

| Field | Bits | Meaning |
|-------|------|---------|
| SOF | 1 | Start of frame — dominant (0), synchronizes all nodes |
| ID | 11 (standard) / 29 (extended) | Message identifier = priority. Lower = higher priority |
| RTR | 1 | Remote Transmission Request — recessive (1) requests data from another node |
| IDE | 1 | Identifier Extension — 0 = standard 11-bit, 1 = extended 29-bit |
| DLC | 4 | Data Length Code — number of data bytes (0–8 for classic, 0–64 for FD) |
| Data | 0–8 bytes | Payload |
| CRC | 15 | Cyclic redundancy check over ID + control + data |
| ACK | 1 | Any receiving node pulls this dominant if CRC passes — sender verifies |
| EOF | 7 | End of frame — 7 recessive bits, marks bus idle |

**Key interview point:** The ACK slot is written recessive by the sender and pulled dominant by any receiver that accepted the frame. If no node pulls it dominant, the sender knows transmission failed.

**Extended vs standard ID:**
- Standard: 11-bit ID (0x000–0x7FF)
- Extended: 29-bit ID — IDE bit set, adds 18 more bits. Your motor controller code used extended IDs (`0x1555AAB0` > `0x7FF`).

---

## Error handling and fault confinement

CAN has hardware-level error detection and a self-healing fault confinement mechanism.

### Error types detected
| Error | What triggers it |
|-------|-----------------|
| CRC error | Receiver CRC doesn't match |
| Bit error | Node reads back a different bit than it sent (except during arbitration/ACK) |
| Stuff error | >5 consecutive bits of same polarity (violates bit stuffing rule) |
| Form error | Fixed-form field (EOF, ACK delim) has wrong polarity |
| ACK error | Sender sees no dominant ACK after transmission |

### TEC / REC counters
Every node maintains two counters:
- **TEC** (Transmit Error Counter) — increments on transmit errors, decrements on success
- **REC** (Receive Error Counter) — increments on receive errors, decrements on success

### Fault confinement states

```
TEC/REC < 128          → Error Active   (normal, sends active error frames)
TEC or REC >= 128      → Error Passive  (sends passive error frames, backs off longer)
TEC >= 256             → Bus-Off        (node disconnects from bus entirely)
```

**Bus-off recovery:** Node waits for 128 × 11 recessive bits, then re-enters Error Active. This is automatic but takes time — a misbehaving node eventually removes itself without manual intervention.

**Why this matters for safety-critical systems:** A single faulty node (e.g., short circuit) can't take down the whole bus — it gets forced into bus-off. This is why CAN is used in medical devices and automotive.

---

## CANopen — the protocol layer

Raw CAN gives you frames with IDs and 8 bytes of data. **CANopen** is a higher-level protocol that standardizes *what those IDs mean* and *how nodes communicate* — used in industrial robotics, medical devices, and motion control.

### CANopen communication objects

| Object | COB-ID pattern | Purpose |
|--------|---------------|---------|
| NMT | 0x000 | Network Management — start/stop/reset nodes |
| SYNC | 0x080 | Synchronization broadcast — triggers simultaneous PDO transmission |
| EMCY | 0x080 + node ID | Emergency — node reports fault |
| PDO (Process Data) | 0x180–0x57F | Real-time data (position, velocity, torque) — no handshake, fast |
| SDO (Service Data) | 0x580–0x5FF | Configuration read/write — confirmed, slower |
| Heartbeat | 0x700 + node ID | Node alive signal |

### PDO vs SDO — the critical distinction

**PDO (Process Data Object):** Fire-and-forget, no acknowledgement. Used for real-time control loops. Pre-configured mapping (which object dictionary entries go in each PDO). Fast — one frame, no response.

**SDO (Service Data Object):** Request/response, confirmed. Used to read/write configuration parameters (gains, modes, limits). Slower — at least 2 frames per transaction.

> Interview answer: "PDOs for real-time data in the control loop — position feedback, torque commands. SDOs for configuration at startup — setting gains, control mode, acceleration limits."

### NMT state machine
```
INITIALISING → PRE-OPERATIONAL → OPERATIONAL
                     ↑                ↓
                     └────── STOPPED ─┘
```
Nodes must be in OPERATIONAL state to send/receive PDOs. NMT master sends a single broadcast frame to transition all nodes simultaneously.

### Object Dictionary
Every CANopen node has an **object dictionary** — a table of parameters addressable by index (16-bit) + subindex (8-bit). Standard indexes defined by CiA (CAN in Automation):
- `0x6040` — controlword (start, halt, fault reset)
- `0x6041` — statusword (current state)
- `0x607A` — target position
- `0x60FF` — target velocity

---

## ROS2 + SocketCAN integration

For WALL-E or any ROS2 robot with CAN devices, `ros2_socketcan` bridges SocketCAN frames into ROS2 topics.

### Setup

```bash
sudo apt install ros-humble-ros2-socketcan
```

### Nodes
- `socketcan_receiver` — reads from CAN interface, publishes `can_msgs/Frame` on `/from_can_bus`
- `socketcan_sender` — subscribes to `can_msgs/Frame` on `/to_can_bus`, writes to CAN interface

### Launch

```python
# in your launch file
Node(
    package='ros2_socketcan',
    executable='socket_can_receiver_node',
    parameters=[{'interface': 'can0', 'interval_sec': 0.01}]
),
Node(
    package='ros2_socketcan',
    executable='socket_can_sender_node',
    parameters=[{'interface': 'can0', 'timeout_sec': 0.01}]
),
```

### Sending a frame from a ROS2 node

```cpp
#include <can_msgs/msg/frame.hpp>

auto pub = create_publisher<can_msgs::msg::Frame>("/to_can_bus", 10);

can_msgs::msg::Frame frame;
frame.id = 0x1555AAB0;
frame.is_extended = true;
frame.dlc = 8;
frame.data = {0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00};
pub->publish(frame);
```

### Receiving and filtering

```cpp
sub_ = create_subscription<can_msgs::msg::Frame>(
    "/from_can_bus", 10,
    [this](const can_msgs::msg::Frame::SharedPtr msg) {
        if (msg->id == 0x1555AAC0) {  // filter for your device's response ID
            // parse msg->data
        }
    });
```

**Tip:** For a CANopen device, layer `lely-core` or `canopen_ros2` on top of `ros2_socketcan` — they handle PDO/SDO/NMT automatically.

---

## Debugging CAN in practice

### candump — sniff all traffic
```bash
candump can0              # all frames
candump can0 -l           # log to file
candump can0 123:7FF      # filter: only IDs 0x123–0x5FF (mask)
candump vcan0             # virtual CAN for dev without hardware
```

Output format: `can0  123   [8]  DE AD BE EF 00 00 00 00`
- Interface, ID (hex), DLC, data bytes

### cansend — inject test frames
```bash
cansend can0 123#DEADBEEF        # standard ID
cansend can0 1FFFFFFF#1122334455 # extended ID (>3 hex digits)
```

### cansniffer — live delta view
```bash
cansniffer can0   # shows only frames that changed — useful for watching motor feedback
```

### canplayer / canlogserver — replay logs
```bash
canplayer -I logfile.log   # replay a recorded session — good for offline debugging
```

### Reading error counters
```bash
ip -details link show can0   # shows TEC, REC, state (error-active/passive/bus-off)
```

Example output when a node is misbehaving:
```
can0: <...> mtu 16 ...
    link/can
    can state ERROR-PASSIVE restart-ms 100
    bitrate 1000000 sample-point 0.750
    tq 25 prop-seg 14 phase-seg1 15 phase-seg2 10 sjw 1
    sja1000: tseg1 1..16 tseg2 1..8 sjw 1..4 brp 1..64 brp-inc 1
    re-started bus-errors arbit-lost error-warn error-pass bus-off
    0          12         3          1          1          0
```

### Common debug checklist
- [ ] 120 Ω termination at both ends of the bus — missing termination = signal reflections = garbage
- [ ] Bitrate matches on all nodes — mismatch = immediate error storm
- [ ] `candump` shows frames — if silent, check `ip link show can0` for bus-off state
- [ ] Extended vs standard ID — sending standard to a device expecting extended (or vice versa) = no ACK = ACK error loop

---

## Interview follow-ups

- **Q:** Why CAN for a motor network instead of, say, Ethernet or UART?
    - **A:** Multi-master with priority-based, non-destructive arbitration gives deterministic latency for critical messages; differential signaling is noise-immune; built-in error confinement. Purpose-built for distributed real-time control in noisy environments.
- **Q:** Two nodes transmit at once — what happens?
    - **A:** Bitwise arbitration: the dominant-bit (lower ID, higher priority) message wins and continues uncorrupted; the loser backs off and retries. No collision destruction, no retransmit penalty for the winner.
- **Q:** Why does ID encode priority?
    - **A:** Because arbitration compares IDs bit by bit and dominant bits win — a numerically lower ID is more dominant, so it has higher priority by construction.
- **Q:** What's CAN FD give you over classic CAN?
    - **A:** Larger payloads (up to 64 bytes) and a faster data phase, while keeping the arbitration scheme.

## CAN FD — Flexible Data-rate

Classic CAN caps at 8 bytes payload and 1 Mbit/s. **CAN FD** (ISO 11898-1:2015) removes both limits while keeping the same arbitration scheme.

### Two phases
| Phase | Speed | Purpose |
|-------|-------|---------|
| Arbitration phase | ≤1 Mbit/s | ID + control bits — slow for reliable multi-master arbitration |
| Data phase | up to 8–12 Mbit/s | Payload + CRC — fast, only one node transmitting |

The speed switches automatically after the arbitration phase completes. This is why it's "flexible" — not a fixed bitrate.

### Key differences from classic CAN
| Feature | Classic CAN | CAN FD |
|---------|-------------|--------|
| Max payload | 8 bytes | 64 bytes |
| Data phase bitrate | Same as arbitration | Up to 8× faster |
| CRC length | 15 bits | 17 or 21 bits (larger payload needs stronger CRC) |
| BRS bit | — | Bit Rate Switch — signals transition to fast data phase |
| ESI bit | — | Error State Indicator — passive/active flag in frame |

### Linux setup for CAN FD
```bash
sudo ip link set can0 type can bitrate 1000000 dbitrate 4000000 fd on
sudo ip link set can0 up
# bitrate = arbitration phase, dbitrate = data phase
```

### When to use CAN FD in robotics
- High-DOF robots sending joint state (position + velocity + torque for 7 joints = 56 bytes — fits in one FD frame, requires 7 classic frames)
- Firmware updates over CAN (large payloads)
- High-bandwidth sensor data (force/torque, IMU at high rates)

---

## Bit timing — configuring for your hardware

When you bring up a CAN interface, the bitrate isn't magic — it's built from smaller time quanta (TQ). Getting this wrong causes intermittent errors even at short distances.

### Anatomy of a CAN bit
```
| Sync Seg | Prop Seg | Phase Seg 1 | Phase Seg 2 |
|    1 TQ  |  1–8 TQ  |   1–8 TQ   |   1–8 TQ   |
                                 ↑
                          sample point
```

| Segment | Purpose |
|---------|---------|
| Sync Seg | Fixed 1 TQ — edge expected here, used for synchronization |
| Prop Seg | Compensates propagation delay (cable length) |
| Phase Seg 1 | Can be lengthened by SJW to resync late edges |
| Phase Seg 2 | Can be shortened by SJW to resync early edges |
| SJW | Synchronization Jump Width — max adjustment per bit for resync |

**Sample point** = end of Phase Seg 1. This is when the bit value is read. Typically 75–87.5% through the bit time.

### Checking current bit timing
```bash
ip -details link show can0
# shows: tq, prop-seg, phase-seg1, phase-seg2, sjw, brp
```

### Rule of thumb for sample point
- 75–80% for noisy environments or long cables
- 87.5% is common default for short, clean buses
- Both ends of the bus must use identical bit timing

---

## CANopen CiA 402 — motion control profile

CiA 402 is the CANopen profile for drives and motion control. Every serious CAN-based motor controller (Maxon EPOS, Elmo, ODrive with CANopen firmware) implements it. Knowing this profile means you can talk to any compliant drive without reading vendor docs from scratch.

### Drive state machine (mandatory)
```
Not Ready → Switch On Disabled → Ready To Switch On → Switched On → Operation Enabled
                                                                           ↓
                                                                      Quick Stop Active
                                                                           ↓
                                                                      Fault Reaction Active → Fault
```

**Controlword (0x6040) commands to transition:**

| Bits [3:0] | Transition |
|-----------|-----------|
| `0xxx` | Shutdown (→ Ready To Switch On) |
| `0111` | Switch On (→ Switched On) |
| `1111` | Enable Operation (→ Operation Enabled) |
| `0000 0010` | Fault Reset |

**Statusword (0x6041) — read to know current state:**
- Bits [6:0] encode the state machine position
- Bit 3: Fault
- Bit 5: Quick Stop active

### Modes of operation (object 0x6060)

| Value | Mode | Use case |
|-------|------|---------|
| 1 | Profile Position | Go to absolute/relative position with trapezoidal profile |
| 3 | Profile Velocity | Run at target velocity |
| 4 | Profile Torque | Output target torque |
| 6 | Homing | Find home position using limit switch or index |
| 8 | Cyclic Sync Position (CSP) | Real-time position commands at fixed cycle (servo loop) |
| 9 | Cyclic Sync Velocity (CSV) | Real-time velocity commands |
| 10 | Cyclic Sync Torque (CST) | Real-time torque commands |

**For robotics control loops:** CSP/CSV/CST (modes 8/9/10) are the right choice — your controller sends a new setpoint every cycle, the drive handles the inner servo loop.

### Startup sequence (every boot)
```
1. NMT: Reset node           → node reinitializes
2. NMT: Enter Pre-Operational → PDO config is possible
3. SDO: Set mode of operation  → write 0x6060
4. SDO: Set PDO mappings       → what goes in each PDO
5. NMT: Enter Operational      → PDOs now active
6. SDO: Controlword = Shutdown → state machine: Ready To Switch On
7. SDO: Controlword = Switch On → Switched On
8. SDO: Controlword = Enable Op → Operation Enabled — drive accepts setpoints
```

### Homing (mode 6) — important for absolute position robots
Before running in position mode, the drive needs to know where zero is:
- Method 1: negative limit switch
- Method 17: positive limit switch
- Method 35: current position is home (software home)

Write homing method to `0x6098`, homing speed to `0x6099`, then set controlword bit 4 to start.

---

## Practical wiring — connectors, cable, stubs

### DB9 pinout (CiA 303-1 standard)
| Pin | Signal |
|-----|--------|
| 2 | CAN_L |
| 3 | GND |
| 7 | CAN_H |
| 9 | CAN_V+ (optional 24V supply) |

Most industrial CAN devices use DB9. Some use bare screw terminals or M12 connectors (field devices).

### Cable
- **Twisted pair** — mandatory. Twisting cancels common-mode noise (both wires pick up same interference, differential receiver rejects it)
- **Shielded twisted pair** — better in motor-heavy environments. Connect shield at one end only to avoid ground loops
- Standard: ISO 11898-2 specifies 120 Ω characteristic impedance

### Stub length limit
Devices tap off the main bus trunk with short stubs. Stubs reflect signals — keep them short:
- At 1 Mbit/s: max stub length ~0.3 m
- At 500 kbit/s: max ~0.6 m
- Rule: stub length < 1/10 of the signal wavelength

**Wrong:**
```
Main bus ────────────────────────────────
              │                │
           [long stub]      [long stub]
              │                │
           Node C            Node D
```

**Right:** daisy-chain topology, nodes tapped directly on the trunk with minimal stub.

### Ground
CAN is differential so theoretically ground-independent — but in practice, excessive ground potential difference between nodes causes common-mode voltage to exceed the receiver's range. Connect a thin ground wire between nodes if they're on separate power supplies.

---

## Hardware filtering — reducing CPU load

On a busy bus (many nodes, high message rate), every frame wakes the CPU if unfiltered. SocketCAN supports hardware and software filters.

### Software filter (SocketCAN)
```c
struct can_filter rfilter[1];
rfilter[0].can_id   = 0x123;
rfilter[0].can_mask = CAN_SFF_MASK;  // 0x7FF — exact match
setsockopt(soc, SOL_CAN_RAW, CAN_RAW_FILTER, &rfilter, sizeof(rfilter));
```

**Mask logic:** `(received_id & mask) == (filter_id & mask)` → accept frame.

### Filter for a range of IDs
```c
rfilter[0].can_id   = 0x580;   // SDO responses start at 0x580
rfilter[0].can_mask = 0x780;   // accept 0x580–0x5FF (all SDO responses)
```

### Disable reception entirely (TX-only node)
```c
int recv_own_msgs = 0;
setsockopt(soc, SOL_CAN_RAW, CAN_RAW_RECV_OWN_MSGS, &recv_own_msgs, sizeof(recv_own_msgs));
```

---

## Node ID and priority assignment strategy

Good ID assignment makes a bus predictable under load. Bad assignment means safety-critical messages get delayed by telemetry.

### Rules
1. **Safety-critical = lowest ID = highest priority.** E-stop, fault broadcast, safety interlock → IDs 0x001–0x00F.
2. **Control loop data = next priority.** Joint commands, velocity setpoints → 0x100–0x1FF.
3. **Sensor feedback = mid priority.** Odometry, force/torque, encoder → 0x200–0x3FF.
4. **Telemetry/logging = lowest priority.** Temperature, diagnostics → 0x600–0x7FF.
5. **Leave gaps** between groups — you'll add nodes later.

### Example assignment for a 3-joint robot arm
| ID | Message |
|----|---------|
| 0x001 | E-stop broadcast |
| 0x010 | NMT (CANopen) |
| 0x101 | Joint 1 position command (PDO) |
| 0x102 | Joint 2 position command (PDO) |
| 0x103 | Joint 3 position command (PDO) |
| 0x201 | Joint 1 encoder feedback (PDO) |
| 0x202 | Joint 2 encoder feedback (PDO) |
| 0x203 | Joint 3 encoder feedback (PDO) |
| 0x601 | Joint 1 temperature telemetry |

---

## EtherCAT vs CAN — when to use which

Roboticists encounter both. Know the tradeoff so you can justify your choice in an interview.

| Feature | CAN / CANopen | EtherCAT |
|---------|--------------|----------|
| Physical layer | Twisted pair, differential | Standard Ethernet cable |
| Topology | Bus (daisy-chain) | Ring (daisy-chain, auto-detected) |
| Max bitrate | 1 Mbit/s (8 Mbit/s FD) | 100 Mbit/s |
| Latency | ~125 µs per frame at 1 Mbit/s | ~100 µs cycle for entire network |
| Payload per cycle | 8–64 bytes per node | Thousands of bytes across all nodes |
| Determinism | Yes (priority arbitration) | Yes (distributed clocks, hardware sync) |
| Hardware cost | Very cheap (every MCU has CAN) | Requires EtherCAT slave chip (ASIC) |
| Complexity | Low | Higher — ESI files, EtherCAT master stack |
| Common use | Cost-sensitive robots, medical, automotive | High-DOF industrial arms, precision motion |

**When to pick CAN:** WALL-E, most hobby/research robots, medical devices, any system where nodes are MCUs (Arduino, STM32) — CAN peripheral is built in, no extra hardware.

**When to pick EtherCAT:** Fanuc, KUKA, Universal Robots, any system needing <1 ms synchronous multi-axis control with high bandwidth. Requires dedicated EtherCAT slave ASICs (e.g., Beckhoff ET1100).

**ROS2 EtherCAT:** `ethercat_driver_ros2` package — same concept as ros2_socketcan but for EtherCAT masters.

---


## Links

- Related: [[Motor Control]], [[Real-Time Determinism]], [[Safety-Critical Architecture]], [[iceoryx Zero-Copy]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

In CAN, what does the message ID encode besides identity? ? Priority — lower ID = higher priority, because bitwise arbitration favors dominant (0) bits.

How is CAN arbitration non-destructive? ? Nodes transmit simultaneously; a node sending recessive that sees dominant backs off. The highest-priority frame continues uncorrupted with no retransmission — unlike Ethernet collision destruction.

What property makes CAN deterministic for critical messages? ? The highest-priority message always wins arbitration immediately and transmits without delay or corruption, giving bounded latency for that frame.

CAN FD vs classic CAN? ? Up to 64 data bytes (vs 8) and a faster data-phase bitrate, keeping the same priority arbitration.

What does the ACK slot do and who writes it? ? Sender transmits it recessive. Any receiver that accepted the frame (CRC passed) pulls it dominant. If no node pulls dominant, sender knows transmission failed and raises an ACK error.

What is bit stuffing in CAN and what error does it catch? ? After 5 consecutive bits of the same polarity, the sender inserts an opposite-polarity bit. Receivers remove it. A stuff error fires if >5 same-polarity bits appear — detects lost synchronization.

CAN fault confinement — three states and their TEC/REC thresholds? ? Error Active (TEC/REC < 128) — normal operation. Error Passive (TEC or REC >= 128) — sends passive error frames, longer backoff. Bus-Off (TEC >= 256) — node disconnects from bus. Recovers after 128 × 11 recessive bits.

PDO vs SDO in CANopen — when do you use each? ? PDO: fire-and-forget, no ack, real-time control loop data (position, torque, velocity). SDO: request/response, confirmed, used for configuration at startup (gains, mode, limits). PDO = fast, SDO = reliable.

What NMT state must a CANopen node be in to send/receive PDOs? ? OPERATIONAL. NMT master sends a broadcast frame to transition all nodes simultaneously.

CANopen object dictionary indexes for position control? ? 0x6040 = controlword, 0x6041 = statusword, 0x607A = target position, 0x60FF = target velocity.

How do you bridge CAN into ROS2? ? ros2_socketcan package — socketcan_receiver publishes can_msgs/Frame on /from_can_bus; socketcan_sender subscribes on /to_can_bus and writes to the CAN interface.

How do you check if a CAN node has gone bus-off? ? `ip -details link show can0` — shows TEC, REC, and state (error-active/passive/bus-off). Also visible as silence on candump.

Two most common causes of a completely silent CAN bus? ? Missing 120 Ω termination at both ends (signal reflections corrupt every frame), or bitrate mismatch between nodes (immediate error storm → bus-off).

Why 120 Ω termination on CAN and why at both ends? ? CAN wire is a transmission line at high bitrates — unterminated ends reflect signals back, corrupting frames. 120 Ω matches the characteristic impedance of standard CAN twisted pair, absorbing the signal. Two resistors because the bus has two physical ends.

Why does CAN max bitrate decrease as cable length increases? ? Longer cable = higher propagation delay. At high bitrates the bit time shrinks — if propagation delay exceeds the bit time, arbitration breaks. 1 Mbit/s → ~40 m max; 125 kbit/s → ~500 m max.

Symptom of missing CAN termination on candump? ? Error frames everywhere, frames randomly missing ACK, TEC climbing. Works at short distances but fails as cable gets longer or bitrate increases.

CAN FD vs classic CAN — the two phases and what changes? ? Arbitration phase runs at ≤1 Mbit/s (same as classic). Data phase runs up to 8 Mbit/s. BRS bit signals the switch. Payload expands to 64 bytes. CRC grows to 17 or 21 bits.

When does CAN FD matter in robotics? ? High-DOF robots (7-joint state in one frame vs 7 frames), firmware updates over CAN, high-rate IMU/force-torque sensor data.

What are the four bit-timing segments in a CAN bit and what does each do? ? Sync Seg (1 TQ, synchronization). Prop Seg (compensates cable propagation delay). Phase Seg 1 (can be lengthened for late edges). Phase Seg 2 (can be shortened for early edges). Sample point = end of Phase Seg 1.

CANopen CiA 402 — what are the three cyclic sync modes and when do you use them? ? CSP (mode 8) = Cyclic Sync Position, CSV (mode 9) = Cyclic Sync Velocity, CST (mode 10) = Cyclic Sync Torque. Use these for real-time control loops — your controller sends a new setpoint every cycle, drive handles inner servo loop.

CiA 402 startup sequence — 8 steps? ? (1) NMT reset. (2) NMT pre-operational. (3) SDO set mode. (4) SDO set PDO mappings. (5) NMT operational. (6) Controlword = Shutdown. (7) Controlword = Switch On. (8) Controlword = Enable Operation.

DB9 CAN pinout (CiA 303-1)? ? Pin 2 = CAN_L, Pin 3 = GND, Pin 7 = CAN_H, Pin 9 = CAN_V+ (optional power).

Why must CAN cable be twisted pair? ? Twisting cancels common-mode noise — both wires pick up the same interference, and the differential receiver rejects it. Untwisted wire acts as an antenna.

Max stub length at 1 Mbit/s and why? ? ~0.3 m. Stubs reflect signals. At high bitrates the bit time is short — a reflection arriving late corrupts the current bit. Rule: stub < 1/10 of signal wavelength.

SocketCAN filter mask logic? ? (received_id & mask) == (filter_id & mask) → accept. To accept exact ID: mask = 0x7FF. To accept a range (e.g., all SDO responses 0x580–0x5FF): id=0x580, mask=0x780.

CAN priority assignment rule for safety-critical systems? ? Safety-critical (e-stop, fault) → lowest IDs (0x001–0x00F). Control loop commands → next (0x100–0x1FF). Sensor feedback → mid (0x200–0x3FF). Telemetry → highest IDs (0x600+). Lower ID = higher priority = guaranteed bus access first.

CAN vs EtherCAT — when to pick each? ? CAN: cheap, built into every MCU, good for research/medical/WALL-E-style robots, up to 1 Mbit/s. EtherCAT: 100 Mbit/s, sub-ms synchronous multi-axis, requires ASIC slave chip — used in Fanuc/KUKA/UR industrial arms.