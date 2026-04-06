# Docker Guide for BlueOS ROS2 + DepthAI
### A Complete Beginner's Guide — Zero Docker Knowledge Required

---

## Part 1 — What is Docker and Why Do We Need It?

### The Problem Without Docker

When you work directly inside the BlueOS terminal and run `apt-get install` or `colcon build`, you are working inside a **Docker container** — a temporary, isolated environment. Every time BlueOS loses power or restarts:

```
Power cuts off
      │
      ▼
Container resets to original image state
      │
      ▼
All your apt installs → GONE
All your builds      → GONE (if outside /root/persistent_ws/)
All your cmake files → GONE
      │
      ▼
Start everything over again 😢
```

This is what kept happening to you. The Pi was also overheating because compiling depthai-ros on a Raspberry Pi is extremely CPU intensive.

### The Solution — Docker Images

A **Docker image** is like a **snapshot or blueprint** of a complete, pre-configured system. Think of it like this:

```
WITHOUT Docker image (what you were doing):
  Every boot → install everything → compile everything → 30-60 min wait

WITH Docker image (what we're doing):
  Build image ONCE on your laptop → push to internet → BlueOS downloads it
  Every boot → just run the image → 0 min wait
  Everything is already compiled and baked in ✅
```

### Key Docker Concepts

| Term | What it means | Real world analogy |
|---|---|---|
| **Image** | A blueprint/snapshot of a complete system | A cake recipe |
| **Container** | A running instance of an image | The actual cake |
| **Dockerfile** | Instructions to build an image | Step-by-step recipe |
| **Layer** | One step in the Dockerfile | One step in the recipe |
| **Registry** | A place to store and share images | A recipe website |
| **Docker Hub / GHCR** | Popular registries | GitHub for Docker images |
| **push** | Upload your image to a registry | Uploading a recipe online |
| **pull** | Download an image from a registry | Downloading a recipe |
| **buildx** | Docker tool for building multi-architecture images | A special oven |

---

## Part 2 — Understanding the Dockerfile Line by Line

A Dockerfile is a text file that contains instructions Docker follows to build your image. Each instruction creates a new **layer** — like floors in a building, each built on top of the previous one.

### The Header

```dockerfile
ARG ROS_DISTRO=jazzy
FROM ros:$ROS_DISTRO-ros-base
```

- `ARG ROS_DISTRO=jazzy` — defines a variable called `ROS_DISTRO` with value `jazzy`. Used throughout the file as `${ROS_DISTRO}`
- `FROM ros:jazzy-ros-base` — **starts from an existing image**. Instead of building from scratch, we start from an official ROS2 Jazzy image that already has ROS2 installed. This is like saying "start with this pre-made base, then add our stuff on top"

```dockerfile
ENV DEBIAN_FRONTEND=noninteractive PIP_BREAK_SYSTEM_PACKAGES=1
```
- `ENV` — sets environment variables inside the container
- `DEBIAN_FRONTEND=noninteractive` — tells apt-get to never ask interactive questions during install (important for automated builds — no one is there to press "yes")

```dockerfile
WORKDIR /root/
```
- Sets the default working directory inside the container. All commands run from `/root/` unless told otherwise

---

### Layer 1 — General Packages (Original, Unchanged)

```dockerfile
RUN rm /var/lib/dpkg/info/libc-bin.* \
    && apt-get clean \
    && apt-get update \
    && apt-get install libc-bin \
    && apt-get install -q -y --no-install-recommends \
    tmux nano nginx wget \
    ros-jazzy-mavros ros-jazzy-mavros-extras \
    ros-jazzy-geographic-msgs \
    ros-jazzy-foxglove-bridge \
    python3-dev python3-pip python3-venv \
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*
```

- `RUN` — executes a shell command inside the container during build
- `&&` — chain multiple commands together (if one fails, stop)
- `\` — line continuation (the command continues on the next line)
- `apt-get update` — refresh the package list
- `apt-get install -q -y --no-install-recommends` — install packages quietly (`-q`), say yes to everything (`-y`), don't install optional extras (`--no-install-recommends`)
- `rm -rf /var/lib/apt/lists/*` — delete the package cache to keep the image small
- Installs: `tmux`, `nginx`, `mavros`, `foxglove-bridge`, Python tools

---

### Layer 2 — GStreamer (Original, Unchanged)

```dockerfile
RUN apt-get update \
    && apt-get install -q -y --no-install-recommends \
    libgstreamer1.0-0 gstreamer1.0-plugins-base ...
```

Installs GStreamer video streaming libraries — needed for the camera driver (`gscam2`). **This layer is unchanged from the original.**

---

### Layer 3 — DepthAI Build Dependencies ⭐ NEW

```dockerfile
RUN apt-get update \
    && apt-get install -q -y --no-install-recommends \
    build-essential \
    cmake \
    ninja-build \
    pkg-config \
    git curl zip unzip tar \
    libbz2-dev libssl-dev zlib1g-dev \
    liblz4-dev liblzma-dev \
    libarchive-dev \
    libusb-1.0-0-dev \
    libprotobuf-dev protobuf-compiler \
    libopencv-dev \
    libspdlog-dev libfmt-dev \
    nlohmann-json3-dev \
    libzstd-dev libbrotli-dev \
    libcurl4-openssl-dev \
    libeigen3-dev \
    libyaml-cpp-dev \
    libwebsocketpp-dev \
    libboost-all-dev \
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*
```

**This is the first new layer we added.** These are all the C++ libraries that depthai-core needs to compile. We discovered each one through the error messages during our manual build attempts on BlueOS. Installing them here in the Dockerfile means they are baked into the image permanently — no more `apt-get install` after every reboot.

| Package | What it's for |
|---|---|
| `build-essential` | C/C++ compiler (gcc, g++, make) |
| `cmake` | Build system generator |
| `ninja-build` | Fast build tool used by cmake |
| `libboost-all-dev` | C++ utility libraries (websocketpp needs this) |
| `libprotobuf-dev` | Protocol Buffers — depthai's data serialization |
| `libopencv-dev` | Computer vision library |
| `libwebsocketpp-dev` | WebSocket library for foxglove streaming |
| `libeigen3-dev` | Linear algebra library |
| `nlohmann-json3-dev` | JSON parsing library |
| `liblz4-dev` `libzstd-dev` | Compression libraries |
| `libusb-1.0-0-dev` | USB device communication (camera connection) |

---

### Layer 4 — CMake Config Files ⭐ NEW

```dockerfile
RUN \
    mkdir -p /usr/lib/aarch64-linux-gnu/cmake/lz4 && \
    printf '...' > /usr/lib/aarch64-linux-gnu/cmake/lz4/lz4Config.cmake && \
    ...
```

**This is the second new layer.** This is the most unusual part of our Dockerfile. Here's why it exists:

Many Ubuntu system libraries (lz4, liblzma, zstd, etc.) are installed correctly but **don't ship with cmake config files** (`.cmake` files). CMake uses these config files to find and link libraries when building software. Without them, CMake says "I can't find lz4" even though lz4 is clearly installed.

We discovered this problem one by one during our manual builds. The fix was to create these config files manually. We're doing the same thing here inside Docker.

```
Without cmake config file:
  Library lz4 IS installed at /usr/lib/aarch64-linux-gnu/liblz4.so
  CMake looks for lz4Config.cmake → NOT FOUND
  CMake error: "Could not find lz4"  ❌

With cmake config file (what we create):
  Library lz4 IS installed at /usr/lib/aarch64-linux-gnu/liblz4.so
  CMake looks for lz4Config.cmake → FOUND at /usr/lib/.../cmake/lz4/
  CMake finds lz4 correctly ✅
```

Libraries that need this fix:

| Library | Why it needs a manual cmake config |
|---|---|
| `lz4` | No cmake config in Ubuntu package |
| `liblzma` | No cmake config in Ubuntu package |
| `zstd` | No cmake config in Ubuntu package |
| `CURL` | Config exists but vcpkg overrides it |
| `httplib` | Header-only, not in apt at all |
| `semver` | Header-only, not in apt at all |
| `magic_enum` | Header-only, not in apt at all |
| `libnop` | Downloaded by depthai as submodule |
| `websocketpp` | No cmake config in Ubuntu package |
| `nlohmann_json` | Config exists but path issue with vcpkg |

---

### Layer 5 — Clone and Build depthai-ros ⭐ NEW

```dockerfile
# First RUN — clone all repositories
RUN mkdir -p /root/camera_ws/src && \
    cd /root/camera_ws/src && \
    git clone --recursive https://github.com/luxonis/depthai-core.git && \
    cd depthai-core && git checkout v3-jazzy && \
    git submodule update --init --recursive && \
    cd .. && \
    git clone https://github.com/luxonis/depthai-ros.git && \
    cd depthai-ros && git checkout v3_humble && cd .. && \
    git clone https://github.com/ros-perception/vision_msgs.git && \
    cd vision_msgs && git checkout ros2 && cd ..
```

**Why 3 repositories?**

```
depthai-core  ← The core C++ library for DepthAI cameras (hardware interface)
    └── includes depthai-core as a submodule

depthai-ros   ← ROS2 wrapper around depthai-core (gives you ROS2 topics/nodes)

vision_msgs   ← Standard ROS2 message types for vision data (needed by depthai-ros)
```

- `git clone --recursive` — clone the repo AND all its submodules (depthai-core has several sub-libraries embedded inside it)
- `git checkout v3-jazzy` — use the branch/version compatible with ROS2 Jazzy
- `git submodule update --init --recursive` — make sure all nested submodules are initialized

**Do we need to include depthai-core separately?** Yes — `depthai-core` is the underlying C++ library. `depthai-ros` is just a ROS2 wrapper around it. Without building `depthai-core` first, `depthai-ros` has nothing to wrap. They are two separate repos and must both be present in `camera_ws/src/`.

```dockerfile
# Second RUN — build depthai_v3 first (the core library)
RUN cd /root/camera_ws && \
    . "/opt/ros/${ROS_DISTRO}/setup.sh" && \
    colcon build \
      --packages-select depthai_v3 \
      --cmake-args \
        -DDEPTHAI_BUILD_EXAMPLES=OFF \
        -DDEPTHAI_BUILD_TESTS=OFF \
        ...
        -DDEPTHAI_OPENCV_SUPPORT=OFF \
        -Dlz4_DIR=/usr/lib/aarch64-linux-gnu/cmake/lz4 \
        ...
```

- `. "/opt/ros/jazzy/setup.sh"` — source ROS2 environment so colcon can find ROS2 tools
- `colcon build` — the ROS2 build tool (like `make` but for ROS2 packages)
- `--packages-select depthai_v3` — only build this one package first (it must succeed before building the rest)
- `--cmake-args` — pass flags to cmake to disable optional features we don't need

**Why disable so many features?**

```
-DDEPTHAI_OPENCV_SUPPORT=OFF    ← We disabled OpenCV integration (not needed for basic use)
-DDEPTHAI_BASALT_SUPPORT=OFF    ← Heavy SLAM library, not needed
-DDEPTHAI_ENABLE_CURL=OFF       ← HTTP logging feature, not needed
-DDEPTHAI_BUILD_EXAMPLES=OFF    ← Example programs, wastes space
-DDEPTHAI_BUILD_TESTS=OFF       ← Test suite, not needed in production
-DDEPTHAI_BUILD_PYTHON=OFF      ← Python bindings, we use C++/ROS2 instead
```

Each `-DXXX=OFF` flag removes a dependency we don't need, making the build faster and simpler.

**Why pass `-Dlz4_DIR=...` etc?** These tell cmake exactly where to find the config files we created in Layer 4. Without these, cmake would search in default locations and still not find them.

```dockerfile
# Third RUN — build everything else
RUN cd /root/camera_ws && \
    . "/opt/ros/${ROS_DISTRO}/setup.sh" && \
    colcon build --packages-skip vision_msgs_rviz_plugins && \
    echo "source /root/camera_ws/install/setup.sh" >> ~/.bashrc
```

- `--packages-skip vision_msgs_rviz_plugins` — skip this package because it needs Qt (GUI display libraries) which are not present on BlueOS (no screen attached)
- `echo "source ..." >> ~/.bashrc` — automatically load the depthai packages every time a terminal opens

---

### Layers 6, 7, 8 — Original Content (Unchanged)

```dockerfile
# Layer 6 — ping-python (sonar driver)
RUN cd /root/ \
    && git clone https://github.com/bluerobotics/ping-python.git -b deployment \
    && cd ping-python \
    && git checkout 3d41ddd \
    && python3 setup.py install --user
```
Installs the BlueRobotics sonar driver. `git checkout 3d41ddd` pins to a specific commit that works with the Ping1D sonar.

```dockerfile
# Layer 7 — original ros2_ws
COPY ros2_ws /root/ros2_ws
RUN cd /root/ros2_ws/ \
    && colcon build --symlink-install \
    ...
```
- `COPY ros2_ws /root/ros2_ws` — copies the `ros2_ws` folder from your laptop into the image
- Builds mavros_control and other original packages

```dockerfile
# Layer 8 — web terminal
ADD files/install-ttyd.sh /install-ttyd.sh
RUN bash /install-ttyd.sh && rm /install-ttyd.sh
COPY files/nginx.conf /etc/nginx/nginx.conf
```
Sets up the web-based terminal you access through the BlueOS browser interface.

---

### Labels

```dockerfile
LABEL version="0.2.0"
LABEL permissions='{ "NetworkMode": "host", ... }'
```

Labels are metadata attached to the image. BlueOS reads the `permissions` label to know how to run the container — network settings, which folders to mount, CPU/memory limits, etc. The `permissions` label is what creates the link between `/usr/blueos/extensions/ros2/` on the host and `/root/persistent_ws/` inside the container.

---

## Part 3 — Step by Step: Build and Deploy

### Prerequisites

```bash
# On your Ubuntu laptop — verify these are installed
docker --version          # needs 20+
docker buildx version     # needs to be installed
git --version
```

### Step 1 — Set Up Multi-Architecture Builder (one time only)

```bash
# This creates a special builder that can build ARM64 images
# on your AMD64 (x86) laptop using emulation
docker buildx create \
  --name multi-arch \
  --platform "linux/arm64,linux/amd64" \
  --driver "docker-container"

# Activate it
docker buildx use multi-arch

# Verify it's active (should show * next to multi-arch)
docker buildx ls
```

**Why do we need this?** Your laptop runs on AMD64 (Intel/AMD chip). BlueROV runs on ARM64 (Raspberry Pi). They are different CPU architectures — code compiled for one won't run on the other. `buildx` uses QEMU emulation to build ARM64 code on your AMD64 laptop.

### Step 2 — Set Up Container Registry (one time only)

You need somewhere to store your built image so BlueOS can download it.

**Option A — Docker Hub (easiest)**
```bash
# 1. Create account at hub.docker.com
# 2. Login
docker login
# enter your Docker Hub username and password
```

**Option B — GitHub Container Registry**
```bash
# 1. Create account at github.com
# 2. Go to: GitHub → Settings → Developer Settings
#    → Personal Access Tokens → Tokens (classic)
#    → Generate new token
#    → Check: write:packages, read:packages
#    → Copy the token

# 3. Login
echo YOUR_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

### Step 3 — Prepare the Dockerfile

```bash
# Clone the original blueos-ros2 repo
git clone --recurse-submodules https://github.com/itskalvik/blueos-ros2
cd blueos-ros2

# Backup original Dockerfile
cp Dockerfile Dockerfile.original

# Replace with our custom Dockerfile
# (copy the Dockerfile from this guide into this folder)
cp ~/Downloads/Dockerfile .

# Verify the folder structure
ls
# Must show: compose.yml  Dockerfile  files  README.md  ros2_ws
```

### Step 4 — Build the Image (runs on your laptop)

```bash
cd ~/blueos-ros2

# For Docker Hub:
docker buildx build \
  --platform linux/arm64 \
  -t YOUR_DOCKERHUB_USERNAME/blueos-ros2-depthai:latest \
  --push \
  .

# For GitHub Container Registry:
docker buildx build \
  --platform linux/arm64 \
  -t ghcr.io/YOUR_GITHUB_USERNAME/blueos-ros2-depthai:latest \
  --push \
  .
```

**What each flag means:**

| Flag | Meaning |
|---|---|
| `--platform linux/arm64` | Build for Raspberry Pi (ARM64) architecture |
| `-t NAME:TAG` | Name and tag your image (`latest` = most recent version) |
| `--push` | Automatically push to registry when done |
| `.` | Use the Dockerfile in the current directory |

**This will take 30-60 minutes** — it's compiling the entire depthai-ros stack inside a Docker container on your laptop. This is the one-time cost. After this, BlueOS just downloads and runs it.

You'll see output like:
```
[1/8] FROM docker.io/library/ros:jazzy-ros-base
[2/8] RUN rm /var/lib/dpkg/info/libc-bin.*...
[3/8] RUN apt-get update && apt-get install...
[4/8] RUN mkdir -p /usr/lib/aarch64...
[5/8] RUN mkdir -p /root/camera_ws/src...   ← git clones here
[6/8] RUN cd /root/camera_ws && colcon...   ← compiling here (slow)
[7/8] RUN cd /root/ && git clone ping...
[8/8] RUN cd /root/ros2_ws/ && colcon...
pushing to registry...
```

### Step 5 — Make Image Public (so BlueOS can pull it)

**Docker Hub:** Go to hub.docker.com → your repo → Settings → Make Public

**GitHub Container Registry:** Go to github.com → Packages → your package → Package Settings → Change visibility → Public

### Step 6 — Deploy to BlueOS

```
1. Open BlueOS in your browser
   → http://192.168.2.2

2. Click the puzzle piece icon (Extensions) in the left panel

3. Click the INSTALLED tab at the top

4. Click the blue + button (bottom right corner)

5. Fill in these fields:
   ┌─────────────────────────────────────────────────┐
   │ Extension Identifier: YourName.ROS2Depthai      │
   │ Extension Name:       ROS2 Depthai              │
   │ Docker image:         YOUR_USERNAME/blueos-ros2-depthai  │
   │ Docker tag:           latest                    │
   └─────────────────────────────────────────────────┘

6. Paste this into the "Original Settings" box:
```

```json
{
  "NetworkMode": "host",
  "HostConfig": {
    "Binds": [
      "/dev:/dev:rw",
      "/usr/blueos/extensions/ros2/:/root/persistent_ws/:rw"
    ],
    "Privileged": true,
    "NetworkMode": "host",
    "CpuQuota": 200000,
    "CpuPeriod": 100000,
    "Memory": 1097152000
  },
  "Env": []
}
```

```
7. Click CREATE

BlueOS will:
  → Pull your image from the registry (~few minutes, one time)
  → Start the container
  → Everything is already compiled and ready
```

### Step 7 — Verify It Works

```bash
# Open the web terminal in BlueOS (left panel)

# Check depthai packages are available
ros2 pkg list | grep depthai

# Expected output:
# depthai_bridge_v3
# depthai_descriptions_v3
# depthai_filters_v3
# depthai_ros_driver_v3
# depthai_ros_msgs_v3
# depthai_ros_v3
# depthai_v3
```

---

## Part 4 — How to Update in the Future

When you need to change anything (add a package, fix a bug, update depthai version):

```bash
# 1. Edit your Dockerfile on your laptop
nano ~/blueos-ros2/Dockerfile

# 2. Rebuild and push
cd ~/blueos-ros2
docker buildx build \
  --platform linux/arm64 \
  -t YOUR_USERNAME/blueos-ros2-depthai:latest \
  --push \
  .

# 3. In BlueOS — restart the extension
# Extensions → your extension → restart button
# BlueOS pulls the new image automatically
```

---

## Part 5 — The Big Picture Summary

```
YOUR LAPTOP (AMD64)                    INTERNET              BLUEROS (ARM64/Pi)
───────────────────                    ────────              ──────────────────

Edit Dockerfile
      │
      ▼
docker buildx build
  (emulates ARM64,              push ──────────► Docker Hub / GHCR
   compiles everything,                               │
   takes 30-60 min)                                   │ pull (one time)
                                                      ▼
                                              BlueOS downloads image
                                                      │
                                                      ▼
                                              docker run (instant)
                                                      │
                                                      ▼
                                              ros2 pkg list ✅
                                              depthai_v3 ✅
                                              No compilation
                                              No apt installs
                                              Survives reboots
```

---

## Part 6 — Why depthai-core Must Be Included

Yes, `depthai-core` must be in the Dockerfile. Here is the dependency chain:

```
depthai-ros (ROS2 nodes, topics, drivers)
      │
      │ depends on
      ▼
depthai-core (C++ library — talks directly to camera hardware)
      │
      │ depends on
      ▼
All the apt libraries (boost, protobuf, lz4, etc.)
```

- `depthai-core` = the engine. It handles USB communication with the camera, runs neural network pipelines, manages data streams.
- `depthai-ros` = the ROS2 steering wheel. It wraps depthai-core and exposes everything as ROS2 topics and nodes so you can use standard ROS2 tools.

Without `depthai-core`, `depthai-ros` cannot compile. They must both be present in `camera_ws/src/` and built in order (`depthai_v3` first, then everything else).
