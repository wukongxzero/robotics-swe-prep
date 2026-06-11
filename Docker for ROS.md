---
type: concept
domain: Compute & acceleration
status: drafted
last-reviewed: 2026-05-27
tags: [compute, docker, containers, ros]
---

# Docker for ROS

> [!question] Explain it cold
> 
> - Why containerize a ROS2 stack at all?
> - What does Docker Compose add over a single container?
> - What's the GPU/device catch when containerizing on a Jetson?

---

## Core idea

Docker packages an app + its exact dependencies into a reproducible image, so "works on my machine" becomes "works everywhere." For ROS2 — which is notoriously dependency-heavy and version-coupled — containers pin the ROS distro, system libs, and Python env, making the stack reproducible across dev machine, robot, and CI.

## Key facts & formulas

- **Image vs container**: image = the built, immutable filesystem+config; container = a running instance. Dockerfile = the recipe.
- **Why for ROS2**: pins the distro (`ros:humble`), avoids polluting the host, makes bring-up reproducible. Pairs with [[colcon ament & launch|colcon]] inside the container.
- **Docker Compose**: declares _multiple_ services and their wiring in one YAML — bring up the whole multi-container system with `docker compose up`. For a robot: one service for ROS2, one for the LLM, networked together.
- **The robotics catches**:
    - **GPU access**: needs the NVIDIA Container Toolkit (`--gpus`/runtime) so the container can reach CUDA — essential on a [[Jetson Orin Setup|Jetson]].
    - **Devices**: serial ports, cameras, CAN — must be passed through (`--device`, privileged, or specific mounts). The Arduino/USB link doesn't exist in the container by default.
    - **Networking & DDS**: [[DDS & RMW|DDS]] discovery needs host networking or proper multicast config, or nodes in different containers won't find each other.
    - **Real-time**: containers add scheduling indirection — fine for perception/LLM, watch latency for hard real-time paths.

## Practical Docker commands

### Daily workflow
```bash
# Build an image from Dockerfile in current directory
docker build -t my_robot:latest .

# Run a container interactively (remove on exit)
docker run -it --rm my_robot:latest bash

# Run with host networking (required for DDS discovery)
docker run -it --rm --network host my_robot:latest bash

# Run with GPU (NVIDIA Container Toolkit required)
docker run -it --rm --gpus all my_robot:latest bash

# Pass through a device (serial port, camera)
docker run -it --rm --device /dev/ttyUSB0 my_robot:latest bash

# Mount host directory into container
docker run -it --rm -v $(pwd)/src:/ros2_ws/src my_robot:latest bash

# Start a stopped container
docker start -i <container_name>

# Exec into a running container
docker exec -it <container_name> bash

# Stop / remove
docker stop <container_name>
docker rm <container_name>

# See running containers
docker ps

# See all containers including stopped
docker ps -a

# See images
docker images

# Remove unused images/containers/volumes
docker system prune
```

### Docker Compose — multi-container robot stack
```yaml
# docker-compose.yml
version: "3.9"
services:
  ros2:
    image: ros:humble
    network_mode: host
    volumes:
      - ./src:/ros2_ws/src
    environment:
      - ROS_DOMAIN_ID=0
    command: bash -c "source /opt/ros/humble/setup.bash && ros2 launch my_pkg my_launch.py"

  llm:
    image: ollama/ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama

volumes:
  ollama_data:
```

```bash
# Bring up the whole stack
docker compose up

# Run in background
docker compose up -d

# See logs from a specific service
docker compose logs -f ros2

# Stop everything
docker compose down

# Rebuild and restart
docker compose up --build
```

### Jetson-specific
```bash
# NVIDIA Container Toolkit — required for GPU/CUDA in containers on Jetson
sudo apt install nvidia-container-toolkit
sudo systemctl restart docker

# Check GPU is visible inside container
docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi

# L4T base image (Jetson-optimized, matches the Jetson Linux version)
docker run -it --rm --runtime nvidia nvcr.io/nvidia/l4t-base:r36.2 bash
```

### Dockerfile for ROS2
```dockerfile
FROM ros:humble

# Install deps
RUN apt-get update && apt-get install -y \
    python3-colcon-common-extensions \
    ros-humble-nav2-bringup \
    && rm -rf /var/lib/apt/lists/*

# Copy and build workspace
WORKDIR /ros2_ws
COPY src/ src/
RUN . /opt/ros/humble/setup.sh && colcon build --symlink-install

# Source overlay on startup
RUN echo "source /ros2_ws/install/setup.bash" >> ~/.bashrc
CMD ["bash"]
```

### Real-time in Docker
```bash
# Give a container real-time scheduling capability
docker run -it --rm \
  --cap-add SYS_NICE \
  --ulimit rtprio=80 \
  my_robot:latest bash

# Inside the container — this will now work
chrt -f 80 ./my_control_loop
```
Note: even with `SYS_NICE`, the container still shares the kernel scheduler. Validate jitter with `cyclictest` inside the container before relying on this for hard real-time.

---


## Interview follow-ups

- **Q:** Why containerize ROS2?
    - **A:** ROS2 is dependency- and version-heavy; a container pins the distro and libs so the stack is reproducible across dev, robot, and CI, without polluting the host. Compose then wires multiple services (ROS2 + LLM) together.
- **Q:** What's special about Docker on a Jetson/robot?
    - **A:** You must pass GPU access (NVIDIA Container Toolkit) and hardware devices (serial, camera, CAN) explicitly, and handle DDS discovery (host networking/multicast) — none of that is automatic, and it's where containerized robots break.
- **Q:** Would you put a hard-real-time control loop in a container?
    - **A:** Cautiously — containers add scheduling indirection. Fine for perception/LLM; for hard real-time I'd validate latency/jitter and consider keeping that path on the host or a tuned runtime.


## Links

- Related: [[colcon ament & launch]], [[Jetson Orin Setup]], [[LLM Tool Calling]], [[DDS & RMW]], [[NixOS for Jetson]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

Why containerize a ROS2 stack? ? ROS2 is dependency/version-heavy; a container pins the distro and libs for reproducibility across dev/robot/CI without polluting the host. Compose wires multiple services (e.g. ROS2 + LLM) together.

Three robotics-specific Docker catches? ? GPU access needs the NVIDIA Container Toolkit; hardware devices (serial/camera/CAN) must be passed through explicitly; DDS discovery needs host networking/multicast or cross-container nodes won't find each other.

How did WALL-E use Docker Compose? ? Two services — ros:humble for the ROS2 nodes and ollama/ollama for the LLaMA server — on the Jetson, keeping heavy dependency sets isolated and the stack reproducible.