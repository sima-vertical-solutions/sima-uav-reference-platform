# SiMa UAV reference platform: what it does, what it needs, where it goes

This document describes the features of the current release, the sensors that unlock each of
them, what a customer gains from starting here, and how the platform is meant to be extended.
For installing see `INSTALL.md` inside a release; for every tunable value see
[`CONFIGURATION.md`](CONFIGURATION.md).

## 1. What the platform is

A complete onboard software stack for an autonomous drone, running on a SiMa.ai Modalix board
under ROS 2 Jazzy. The board does the perception and odometry work on its machine learning
accelerator (MLA), at sensor rate, in the power budget of a small airframe, and hands the flight
controller what it needs to fly. The release is installable as-is on a flashed board, and every
part of it is configuration-driven so it can be adapted rather than rewritten.

The release is a reference, not a product: it shows one working, measured integration of
camera, radar and flight controller, and it is built so that each of those can be swapped.

## 2. Features in release 0.4.2

### Camera capture

One node owns the MIPI camera and publishes it at 1920x1080, 30 frames per second, in the NV12
format the accelerator consumes natively. Nothing converts colour on the CPU. The same frames feed
both perception and odometry, so the two never contend for the device. Camera intrinsics are
published alongside every frame.

### On-device segmentation

A YOLOv8 instance-segmentation model runs on the MLA at the camera's full rate, about 30 Hz,
producing class, bounding box and mask per detection. Thresholds, class limits and the model
archive itself are configuration.

### Radar and camera fusion

Detections from the camera are matched with points from a TI IWR6843ISK millimetre-wave radar.
Each fused detection carries a range and a radial velocity, gated by the segmentation mask so the
range belongs to the object and not its background. Output is at the radar's rate, 20 Hz, up to
the radar's physical limit of about 11 m. Every detection says whether radar contributed, so a
consumer can tell "no radar on this vehicle" from "radar saw nothing".

### Visual odometry

SuperPoint feature extraction and LightGlue matching both run on the MLA, around 38 ms per frame
pair. A keyframe-based motion estimator turns matches into a pose delta, with metric scale taken
from the flight controller's height and attitude. The node speaks MAVLink and is designed to
feed ArduPilot's EKF3 as a vision position source for GPS-denied flight. It publishes a
dead-reckoned odometry topic for bench checks and recording.

The link to the flight controller ships **disabled by configuration** and is enabled per vehicle
once three airframe-specific values have been measured (section 6). In simulation it is on, and it
includes a visual-inertial mode that fuses the simulated IMU and rangefinder.

### Simulation with the same stack

The release ships the simulation side: Isaac Sim scenes of an urban environment in several
variants, a quadrotor model, an ArduPilot SITL bridge, and publishers that deliver frames and
sensor data over ROS 2. The board runs exactly the same perception and odometry code against the
simulator as against the sensors. Only a handful of configuration values change, and the install
flag `--mode sim` applies them. This is how new scenes, flight patterns and model variants are
evaluated before they reach an airframe.

### Operations and verification

- **Health check before install.** `provision.sh` reports the flight controller, radar, camera
  (by capturing a frame, not just listing the device), the ROS 2 base, the accelerator runtime
  version and the MAVLink bridge, and says what each gap costs.
- **Install, update, roll back.** `deploy.sh` ships the overlay, resolves the camera identity on
  the board, bakes the hardware declaration into the run wrappers and verifies integrity with
  SHA-256 before anything is copied. Rolling back is deploying the previous folder.
- **Acceptance gate.** A rate checker measures every topic against the rates that were measured on
  the reference vehicle, with a profile per hardware configuration, so "it launched" and "it runs
  at speed" are different results.
- **Operator view.** A terminal UI shows each stack's status and logs with restart and stop keys.
  A Foxglove bridge with shipped layouts gives a graphical view of detections, radar and odometry.
- **Pipeline health check** watches the topic chain for staleness. A **rosbag allowlist** records
  the radar, detection and odometry topics for later analysis.
- **Traceability.** Every release carries a version stamp with the exact source, platform and
  model checksums it was built from, and the board reports what it runs.

### Board bring-up documentation

Step-by-step guides for radar firmware flashing, the MAVLink serial-to-UDP bridge, a WiFi hotspot
on the board, an LTE modem and the Foxglove bridge. These are the one-time steps a release does
not automate.

## 3. Sensors and hardware

### The reference set

| component | reference part | interface | what it enables |
|---|---|---|---|
| compute | SiMa.ai Modalix DevKit, flashed base image | | everything; the MLA runs all three models |
| camera | MIPI camera, Sony IMX568 class, 1920x1080 at 30 fps | MIPI CSI | segmentation, fusion, visual odometry |
| radar | TI IWR6843ISK, small-obstacle-detection firmware | USB (two serial ports) | range and velocity on detections |
| flight controller | CubeOrange+, ArduPilot, EKF3 | USB or UART, MAVLink 2 at 921600 baud | metric scale for odometry; the consumer of the vision pose |
| MAVLink bridge | mavlink-router or MAVProxy on the board | | carries UDP from the odometry node to the controller's serial link, and state back |
| network | USB-C Ethernet, WiFi hotspot or LTE | | installing, operating and observing the board |

### What each sensor adds, and what its absence costs

Only the camera is a hard requirement; the installer refuses to run without one. Everything else
degrades gracefully and is declared at install time so the health check and acceptance gate judge
the vehicle you actually have.

| configuration | install flags | you get | you lose |
|---|---|---|---|
| camera + radar + flight controller | none | the full stack: ranged detections, odometry with metric scale, vision pose to the controller | nothing |
| camera + flight controller | `--no-radar` | segmentation at 30 Hz, odometry | range and velocity on detections; the fusion output is silent |
| camera only | `--no-radar --no-fc` | segmentation, odometry as a debug topic | metric scale and any effect on flight |
| no sensors, replaying a recording | `--no-camera --no-radar` | the whole stack exercised from a bag | live operation |
| simulation | `--mode sim` | perception and visual-inertial odometry against Isaac Sim | nothing simulates the radar |

For optimal use, fit all three. The radar is what turns a segmentation mask into an obstacle at a
known distance and closing speed, and the flight controller is what turns a visual pose delta into
metres and into flight.

### Off-board

- A laptop with ssh for installing and for Foxglove.
- For simulation, a workstation with an NVIDIA GPU running the Isaac Sim 6.1 container.

## 4. What a customer gains

- **Time to first result.** A flashed board becomes a perceiving, odometry-producing drone computer
  in one install with no build step. The health check tells you up front what the board is missing.
- **Perception on the edge, in the power budget.** Segmentation, feature extraction and matching
  all run on the MLA at sensor rate. No GPU, no ground link, nothing that stops working when the
  radio does.
- **GPS-denied flight path.** The odometry chain to the flight controller is in place and
  configuration-gated, with simulation evidence behind it, so the remaining work is measurement on
  your airframe rather than integration.
- **One stack for simulation and hardware.** Scenes, variants and models are evaluated in Isaac Sim
  with the same code and the same acceptance gate that run on the vehicle.
- **A release process you can trust.** Checksums at every step, model identity verified against
  the manifest, measured acceptance rates per configuration, versioned and reversible installs.
- **Honesty about the unknowns.** Every value not yet verified on the reference vehicle is marked
  as such and collected in one list, so adaptation starts from facts rather than from guesses.

## 5. Extending the platform

The stack is a thin client over a shared library of capabilities. The client holds the selection
of capabilities, the configuration and the models; the capabilities hold the code. Extensions
fall into four tiers, cheapest first.

### Tune: change a value

Thresholds, topics, rates, range gates, mount angles, intrinsics, the MAVLink endpoint. All in the
YAML files under the configuration tree, documented in `CONFIGURATION.md`. On the board, edit the
installed copy and restart to try a value; put it in the client repository to keep it.

### Swap: a different model or sensor

- **Models.** Any NEAT MPK archive with a supported decode family can replace the segmentation
  model by changing `model_path` and `decode_type`; the inference node supports YOLOv8 detection,
  segmentation and pose, SSD and others. The two LightGlue variants shipped are selected the same
  way. Model identity is pinned by checksum in the manifest.
- **Camera.** The camera name is resolved on the board at install, so a different MIPI module needs
  its intrinsics and resolution in configuration and nothing else.
- **Radar.** A different radar profile is a different `.cfg`; the fusion node consumes any
  PointCloud2 with an optional Doppler field. A different radar is a different driver publishing
  that message.
- **Flight controller.** The odometry node speaks MAVLink over UDP; any controller that accepts
  vision position deltas works behind the bridge.

### Add: a new capability

A new stack is a launch include plus one dependency line in the bring-up package. Capabilities
live in the shared library and are selected by package name, so a client picks what it needs and
nothing else is built. The rover built on the same library uses RTAB-Map SLAM and Nav2 where this
drone uses monocular odometry; both are selections, not forks.

### Simulate: a new scene or vehicle

Scenes are built from a base city with variant generators for lighting, obstacles and moving
people. A new airframe is a new USD asset and a motor model. Flight patterns are scripted and
produce reports, so a change is measured before it is flown.

## 6. Before you fly: vehicle-specific calibration

A reference platform cannot ship values that belong to one airframe. Three of them decide whether
the vision pose the flight controller receives is right, and a confidently wrong pose is worse for
an EKF than none, so the odometry-to-controller link ships off and is turned on per vehicle after
these are measured and checked on the bench with the aircraft restrained.

| value | config key | what it is | how to get it |
|---|---|---|---|
| camera mount | `motion.cam_to_body_roll/pitch/yaw` | rotation from the camera's optical frame to the airframe's body frame. Shipped as forward-facing with zero tilt | from the mechanical mount, or a short calibration flight comparing VO deltas against the controller's own motion |
| height offset | `mavlink.agl_offset_m` | height of the camera lens above the point the controller calls ground at arming. Metric scale is derived from the controller's height, so this sets the scale at low altitude | a tape measure on the landed vehicle |
| bridge endpoint | `mavlink.target_ip` / `target_port` | the UDP address the MAVLink bridge on the board listens on; the odometry node sends there and receives attitude and height back on the same socket | from the mavlink-router or MAVProxy configuration on the board; confirm with a heartbeat |

Once the three are in `config/vo/visual_odometry.yaml`, enabling the link is removing two lines
from the hardware half, `visual_odometry.hw.yaml`: the `scale_mode: parallax` override and
`mavlink.enable: false`. Then set ArduPilot's `VISO_DELAY_MS` to match keyframe emission (155 ms)
and verify the pose stream from the ground station before arming.

Two more values are per vehicle and already filled for the reference airframe. Re-measure them
for yours:

- **Camera intrinsics** (`fx fy cx cy` and distortion), in `camera.yaml` and repeated under
  `visual_odometry.motion.camera`. A standard checkerboard calibration at the camera's native
  resolution.
- **Radar-to-camera extrinsic**, published as a static transform. Measured on the mount; the fusion
  node's `debug_projection` output exists to check it visually.

And one setting that is not calibration but is still yours to change: the WiFi hotspot's SSID
and passphrase in `provisioning/wifi/`. The release ships a default. Never fly on it.

## 7. Known limitations in this release

- Radar profile selection from the launch is not wired; the active profile is whatever the driver's
  installed share holds, and a reflash resets it.
- The vehicle description (URDF) does not exist yet, so there is no TF tree beyond the sensors.
- The alternative LightGlue variant is shipped for a different accelerator runtime build and is not
  loaded here.
