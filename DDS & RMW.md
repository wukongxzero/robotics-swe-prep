---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: tags: [ros2, dds, rmw, middleware]

# DDS & RMW


---

## Core idea

**DDS** (Data Distribution Service) is the industry pub/sub middleware standard ROS2 sits on — decentralized, no master, with built-in QoS. **RMW** (ROS Middleware) is the abstraction layer that lets ROS2 talk to _any_ DDS vendor through a common interface, so you can swap implementations without touching application code.

## Key facts & formulas

- **No master**: unlike ROS1's centralized `roscore`, DDS nodes discover each other peer-to-peer. Removes a single point of failure.
- **Discovery**: participants announce themselves (simple/SPDP for participants, SEDP for endpoints), typically via multicast on a domain. Everyone learns everyone's topics/QoS. Downside: **discovery traffic grows with node count** — large fleets need discovery servers or partitioning.
- **Domain ID**: isolates DDS traffic — different domain IDs don't see each other. Use it to separate robots/test setups on one network.
- **RMW implementations**: Fast DDS (eProsima, common default), Cyclone DDS, Connext (RTI, used in regulated/industrial). Swap via `RMW_IMPLEMENTATION` env var.
- QoS lives at the DDS layer — see [[ROS2 QoS]]. That's _why_ QoS feels low-level: it's the DDS contract surfaced up.
- Transport: typically UDP under the hood; loopback/shared-memory transports exist per-vendor (lead-in to [[iceoryx Zero-Copy]]).


## Links

- Related: [[ROS2 QoS]], [[iceoryx Zero-Copy]], [[Latency Budgets]]
- Parent: [[00 Knowledge Map]]

---

