---
type: concept
domain: ROS2 & middleware
status: drafted
last-reviewed: 2026-06-02
tags: [ros2, multi-machine, dds, network, distributed, jetson]
---

# Multi-Machine ROS2


---

## How it works by default

ROS2 uses DDS (Data Distribution Service) for discovery and communication. On a local network, DDS **auto-discovers** other nodes via multicast — no ROS master needed (unlike ROS1).

Two machines on the same network with the same `ROS_DOMAIN_ID` will automatically see each other's topics, services, and actions.

```
Laptop (ROS2 Jazzy)  ←──── LAN ────→  Jetson (ROS2 Humble)
  ros2 topic list                        ros2 topic list
  # sees Jetson topics                   # sees Laptop topics
```

---

## ROS_DOMAIN_ID

Isolates ROS2 networks. Nodes with different domain IDs cannot communicate.

```bash
# Default is 0 — everyone on the network sees each other
export ROS_DOMAIN_ID=42   # isolate to your team's robots

# Add to ~/.bashrc to persist
echo "export ROS_DOMAIN_ID=42" >> ~/.bashrc
```

**When to change:** multiple robots on the same network, multiple teams, or CI systems that shouldn't interfere with each other.

---

## WALL-E hardware setup (Laptop ↔ Jetson)

```bash
# On Jetson (~/.bashrc)
export ROS_DOMAIN_ID=10
source /opt/ros/humble/setup.bash
source ~/WALL_E/src/install/setup.bash

# On Laptop (~/.bashrc)
export ROS_DOMAIN_ID=10   # same domain
source /opt/ros/jazzy/setup.bash

# Connect laptop to same WiFi/Ethernet as Jetson
# Then:
ros2 topic list              # should see Jetson topics
ros2 topic echo /odom        # reads from Jetson
```

**Cross-version (Humble ↔ Jazzy):** works if message types are compatible (they usually are for standard msgs). Custom interfaces must be compiled on both machines.

---

## Network requirements for DDS

DDS uses **multicast UDP** for discovery (239.255.0.1, port 7400 by default).

```bash
# Check multicast is enabled on the interface
ip maddr show | grep 239.255

# If multicast blocked (some routers block it):
export RMW_FASTRTPS_USE_UDP_ONLY=1  # disable multicast, use unicast
# OR configure DDS peers manually (see below)
```

---

## Manual peer configuration (when multicast fails)

```xml
<!-- fastdds_config.xml -->
<DomainParticipantFactory>
  <participants>
    <participant>
      <rtps>
        <builtin>
          <metatrafficUnicastLocatorList>
            <locator>
              <udpv4><address>192.168.1.100</address></udpv4>  <!-- Jetson IP -->
            </locator>
          </metatrafficUnicastLocatorList>
        </builtin>
      </rtps>
    </participant>
  </participants>
</DomainParticipantFactory>
```

```bash
export FASTRTPS_DEFAULT_PROFILES_FILE=/path/to/fastdds_config.xml
```

---

## Bandwidth considerations

Large topics (images, point clouds) can saturate WiFi:

| Topic | Approximate bandwidth |
|-------|-----------------------|
| `/camera/color/image_raw` (640x480 RGB, 30Hz) | ~265 Mbps |
| `/scan` (LaserScan, 40Hz) | ~1 Mbps |
| `/odom` (50Hz) | ~0.5 Mbps |
| `/map` (occupancy grid, 1Hz) | ~2 Mbps |

**Solutions:**
- Compress images: `image_transport` with `compressed` or `theora` transport
- Reduce camera rate for remote monitoring
- Run heavy processing ON the Jetson, only send results over network

---

## Debugging multi-machine issues

```bash
# Check domain IDs match
printenv ROS_DOMAIN_ID

# Check both machines see each other
ros2 node list    # should show nodes from both machines

# If nodes not discovered:
# 1. Check same domain ID
# 2. Check same network (ping each other)
# 3. Check firewall (ufw allow 7400:7500/udp)
# 4. Check multicast or switch to unicast

# Check topic bandwidth
ros2 topic bw /camera/color/image_raw

# Check which machine a node is on
ros2 node info /node_name  # shows hostname
```

---

## Firewall rules for ROS2 (Ubuntu)

```bash
sudo ufw allow 7400:7500/udp   # DDS discovery
sudo ufw allow 7000:7100/udp   # DDS data
```

---

## Links
- Related: [[DDS & RMW]], [[ROS2 QoS]], [[Jetson Orin Setup]], [[WALL-E V3]]
- Parent: [[00 Knowledge Map]]

---

