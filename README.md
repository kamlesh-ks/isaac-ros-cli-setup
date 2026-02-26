# Bake GEM Packages into a Custom Docker Layer (Step-by-Step)

This is the recommended approach for reproducible, team-shareable environments.

When you install `isaac-ros-cli` via `sudo apt-get install isaac-ros-cli`, the
Debian package ships everything the build system needs:

```
/etc/isaac-ros-cli/
├── .build_image_layers.yaml          # Build layer ordering
├── .isaac_ros_common-config          # Contains: CONFIG_DOCKER_SEARCH_DIRS=(docker)
├── .isaac_ros_dev-dockerargs         # Extra docker run args
├── environment.conf                  # ISAAC_ROS_ENVIRONMENT=docker
├── docker/                           # <-- All Dockerfiles are HERE
│   ├── Dockerfile.isaac_ros
│   ├── Dockerfile.noble
│   ├── Dockerfile.ros2_jazzy
│   ├── Dockerfile.realsense
│   ├── Dockerfile.zed
│   ├── rosdep/extra_rosdeps.yaml
│   ├── scripts/
│   └── ...
/usr/lib/isaac-ros-cli/
├── run_dev.py                        # Container lifecycle
├── build_image_layers.py             # Layered Docker build engine
└── isaac_ros_common_config_utils.py  # Config resolution
/usr/share/isaac-ros-cli/
└── config.yaml                       # Default config (read-only)
```

The `.isaac_ros_common-config` file tells the build system `CONFIG_DOCKER_SEARCH_DIRS=(docker)`,
which resolves to `/etc/isaac-ros-cli/docker/`. That is where it looks for
`Dockerfile.<key>` files. Your custom Dockerfile goes in the same directory.

## Full Walkthrough: AprilTag as Example

**Step 1: Verify your install**

```bash
# Confirm isaac-ros-cli is installed
isaac-ros --help

# Confirm the Dockerfiles are in place
ls /etc/isaac-ros-cli/docker/Dockerfile.*
# Expected output:
#   /etc/isaac-ros-cli/docker/Dockerfile.isaac_ros
#   /etc/isaac-ros-cli/docker/Dockerfile.noble
#   /etc/isaac-ros-cli/docker/Dockerfile.ros2_jazzy
#   /etc/isaac-ros-cli/docker/Dockerfile.realsense
#   /etc/isaac-ros-cli/docker/Dockerfile.zed
```

**Step 2: Initialize Docker mode (one-time)**

```bash
sudo isaac-ros init docker
```

**Step 3: Set your workspace**

```bash
mkdir -p ~/workspaces/isaac_ros-dev
export ISAAC_ROS_WS=~/workspaces/isaac_ros-dev

# Make it permanent
echo 'export ISAAC_ROS_WS=~/workspaces/isaac_ros-dev' >> ~/.bashrc
```

**Step 4: Create your custom Dockerfile**

Place it in `/etc/isaac-ros-cli/docker/` so the build system finds it automatically:

```bash
sudo tee /etc/isaac-ros-cli/docker/Dockerfile.apriltag_gem << 'DOCKERFILE'
ARG BASE_IMAGE=ubuntu:24.04
FROM ${BASE_IMAGE}

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    apt-get update && apt-get install -y \
        ros-jazzy-isaac-ros-apriltag \
        ros-jazzy-isaac-ros-apriltag-interfaces
DOCKERFILE
```

That's the entire Dockerfile. It takes the previous layer's image as `BASE_IMAGE`
(injected automatically by the build system) and installs the AprilTag GEM on top.

**Step 5: Tell the CLI to use your new layer**

Create a user-level config override:

```bash
mkdir -p ~/.config/isaac-ros-cli

cat > ~/.config/isaac-ros-cli/config.yaml << 'EOF'
version: 2

docker:
  image:
    base_image_keys:
      - isaac_ros
    additional_image_keys:
      - apriltag_gem
  run:
    container_name: isaac_ros_apriltag_dev
EOF
```

**Step 6: Update the build layer order**

The build system needs to know where `apriltag_gem` fits in the chain.
Edit the system build config to add your key:

```bash
sudo tee /etc/isaac-ros-cli/.build_image_layers.yaml << 'EOF'
image_key_order:
  - isaac_ros.noble.ros2_jazzy.realsense.nova_carter.apriltag_gem
context_overrides:
  isaac_ros: ..
  noble: ..
cache_to_registry_names: []
cache_from_registry_names:
  - nvcr.io/nvidia/isaac/ros
remote_builder: []
EOF
```

The only change is appending `.apriltag_gem` to the `image_key_order` list.
This tells the build system that `apriltag_gem` comes after the other layers.

**Step 7: Build and activate**

```bash
isaac-ros activate --build-local --verbose
```

What happens under the hood:
1. CLI reads your config: keys = `[isaac_ros, noble, ros2_jazzy, apriltag_gem]`
2. Build system searches `/etc/isaac-ros-cli/docker/` for each `Dockerfile.<key>`
3. Builds the chain:
   - `Dockerfile.isaac_ros` (Ubuntu Noble + ROS 2 base + apt repos)
   - `Dockerfile.noble` (CUDA + PyTorch + dev libs) -- receives output of previous as `BASE_IMAGE`
   - `Dockerfile.ros2_jazzy` (full ROS 2 Jazzy) -- receives output of previous as `BASE_IMAGE`
   - `Dockerfile.apriltag_gem` (your layer) -- receives output of previous as `BASE_IMAGE`
4. Final image is cached locally
5. Container starts with your workspace mounted at `/workspaces/isaac_ros-dev`

**Step 8: Verify inside the container**

```bash
# You're now inside the container
ros2 pkg list | grep apriltag
# Expected output:
#   isaac_ros_apriltag
#   isaac_ros_apriltag_interfaces

# Run a quick test
ros2 launch isaac_ros_apriltag isaac_ros_apriltag.launch.py
```

## Subsequent runs

After the first build, the image is cached. Future activations are instant:

```bash
isaac-ros activate
# Attaches to existing container, or starts a new one from the cached image
```

To rebuild after changing the Dockerfile:

```bash
isaac-ros activate --build-local --no-cache --verbose
```