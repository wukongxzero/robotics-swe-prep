---

## type: concept domain: ROS2 & middleware status: drafted last-reviewed: tags: [ros2, dds, rmw, middleware]

# DDS & RMW

> [!question] Explain it cold
> 
> - What is DDS and why did ROS2 build on it (vs ROS1's master)?
> - What is the RMW layer and why does it exist?
> - How does discovery work, and what's the downside at scale?

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

## Where I've used it

- **Articulus**: ran DDS with **iceoryx** for zero-copy shared-memory transport on the latency-critical path — DDS for discovery/structure, iceoryx for the actual high-rate, low-latency data movement. Vendor choice and transport tuning were real engineering decisions, not defaults.
- Awareness that **Connext** shows up in regulated medical/industrial contexts is useful framing for surgical-robotics interviews.

## Interview follow-ups

- **Q:** Why did ROS2 drop the master?
    - **A:** ROS1's `roscore` was a single point of failure and a bottleneck. DDS gives decentralized peer discovery, built-in QoS, and a real-time-capable, standardized transport — better fit for distributed, safety-critical systems.
- **Q:** What does the RMW layer give you?
    - **A:** Vendor independence — application code targets the ROS middleware interface, and you select Fast DDS / Cyclone / Connext underneath without code changes. Lets you pick based on latency, licensing, or certification needs.
- **Q:** Discovery problem at scale?
    - **A:** Default multicast discovery is O(N²)-ish in chatter as participants grow. Mitigate with discovery servers, domain partitioning, or restricting discovery ranges.

## Gotchas / what trips me up

- Saying "ROS2 has no networking config to worry about" — domain IDs, multicast reachability, and RMW choice all bite in practice.
- Conflating DDS (the standard/transport) with RMW (the ROS abstraction over it).

## Links

- Related: [[ROS2 QoS]], [[iceoryx Zero-Copy]], [[Latency Budgets]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

What does the RMW layer abstract? ? The specific DDS vendor implementation — app code targets the ROS middleware interface, letting you swap Fast DDS / Cyclone / Connext without code changes.

Why did ROS2 move from a master (roscore) to DDS? ? DDS is decentralized (peer discovery, no single point of failure), has built-in QoS, and is a real-time-capable standardized transport.

What's the scaling problem with default DDS discovery? ? Multicast discovery chatter grows roughly quadratically with participant count; large systems need discovery servers or domain partitioning.

What does the DDS domain ID do? ? Isolates traffic — participants on different domain IDs don't discover or communicate with each other.