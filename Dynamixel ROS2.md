---

## type: concept domain: Real-Time & Embedded status: drafted last-reviewed: tags: [dynamixel, ros2, serial, actuator, protocol2, cpp]

# Dynamixel ROS2


---

## Core idea

Dynamixel motors (XM, XH series) communicate over a half-duplex TTL/RS485 serial bus using ROBOTIS Protocol 2.0. Each motor has an ID and a **control table** — a memory map of registers you read/write to control behavior. The SDK abstracts this into `PacketHandler` (protocol layer) and `PortHandler` (serial layer). Bulk read/write lets you read/write different registers on different motors in a single bus transaction — critical for multi-joint arms.

---

## Control table (Protocol 2.0 key addresses)

| Address | Name | Size | Notes |
|---------|------|------|-------|
| 11 | OPERATING_MODE | 1 byte | 3=position, 1=velocity, 16=PWM |
| 64 | TORQUE_ENABLE | 1 byte | 0=off, 1=on — must enable before motion |
| 65 | LED | 1 byte | debug use |
| 116 | GOAL_POSITION | 4 bytes | target position in ticks (0–4095 for 360°) |
| 132 | PRESENT_POSITION | 4 bytes | current position in ticks |

**Position conversion:**
```
ticks = (degrees / 360.0) * 4096   → for XM430 (0–4095 range)
degrees = (ticks / 4096.0) * 360.0
radians = (ticks / 4096.0) * 2π
```

---

## Setup sequence (must follow this order)

```cpp
// 1. Open port
portHandler->openPort();           // opens /dev/ttyUSB1

// 2. Set baud rate
portHandler->setBaudRate(57600);   // must match motor baud DIP/config

// 3. Set operating mode BEFORE enabling torque
packetHandler->write1ByteTxRx(portHandler, DXL_ID, ADDR_OPERATING_MODE, 3, &error);
// mode 3 = position control

// 4. Enable torque LAST — locks config registers
packetHandler->write1ByteTxRx(portHandler, DXL_ID, ADDR_TORQUE_ENABLE, 1, &error);
```

**Critical rule:** Operating mode can only be changed when torque is **disabled**. Changing mode with torque on is silently ignored.

---

## Single motor read/write

```cpp
// Write goal position (4 bytes)
packetHandler->write4ByteTxRx(
    portHandler, dxl_id, ADDR_GOAL_POSITION, position, &dxl_error);

// Read present position (4 bytes)
int32_t position;
packetHandler->read4ByteTxRx(
    portHandler, dxl_id, ADDR_PRESENT_POSITION, (uint32_t*)&position, &dxl_error);
```

---

## Bulk read/write (multi-motor, different registers)

**Why bulk?** Each individual read/write is a full serial transaction (TX packet → wait → RX packet). For a 6-DOF arm reading 6 positions one at a time: 6× latency. Bulk read/write batches them into one transaction.

- **Sync write:** same register, same data size, multiple motors — one TX, no RX
- **Bulk write:** different registers, different data sizes per motor — one TX, no RX
- **Sync read:** same register, multiple motors — one TX, one RX per motor (but faster than individual)
- **Bulk read:** different registers per motor — one TX, one RX per motor

**From the code (4 motors: IDs 1, 3, 5, 10):**
```cpp
// ROS2 service to read all 4 positions at once
void bulkGetItemCallback(req, res) {
    res->value1 = getPosition(req->id1);  // ID 1
    res->value2 = getPosition(req->id2);  // ID 3
    res->value3 = getPosition(req->id3);  // ID 5
    res->value4 = getPosition(req->id4);  // ID 10
}

// ROS2 subscription to set all 4 positions at once
void bulkSetItemCallback(msg) {
    setPosition(msg->id1, msg->value1);
    setPosition(msg->id2, msg->value2);
    setPosition(msg->id3, msg->value3);
    setPosition(msg->id4, msg->value4);
}
```

---

## ROS2 interface design

The node exposes:
- **Service** `/bulk_get_item` → request: 4 motor IDs → response: 4 positions
- **Subscription** `/bulk_set_item` → message: 4 (id, position) pairs

**Why service for read, subscription for write?**
- Reads need a response — service is request/reply
- Writes are fire-and-forget — subscription is one-way, lower overhead

**Custom interfaces** (`dynamixel_sdk_custom_interfaces`):
```
BulkSetItem.msg:   uint8 id1, int32 value1, uint8 id2 ... (×4)
BulkGetItem.srv:   uint8 id1..4 → int32 value1..4
```

---

## Hardware wiring

```
Jetson / PC
    │  USB
    ▼
USB2Dynamixel / U2D2 (USB ↔ TTL/RS485 adapter)
    │  TTL half-duplex (3-wire: GND, DATA, VDD)
    ▼
Dynamixel motor chain (daisy-chained, each with unique ID)
    ID 1 → ID 3 → ID 5 → ID 10
```

**Half-duplex:** TX and RX share the same data line — SDK handles the direction switching internally. Don't use a standard full-duplex UART adapter.

---

## Common failure modes

| Problem | Cause | Fix |
|---------|-------|-----|
| Motor doesn't move | Torque not enabled | Check ADDR_TORQUE_ENABLE = 1 |
| Mode change ignored | Torque was enabled | Disable torque → change mode → re-enable |
| Communication timeout | Baud mismatch | Use ROBOTIS wizard to check/set motor baud |
| Position jumps on enable | Goal position ≠ current position | Read current position first, write it as goal before enabling |
| Half-duplex conflict | Wrong adapter | Must use U2D2 or USB2Dynamixel, not generic USB-serial |

---

## Links

- Related: [[Serial Packet Protocols]], [[ROS2 Node Design Patterns]], [[Motor Control]], [[CAN Bus]]
- Parent: [[00 Knowledge Map]]

---

