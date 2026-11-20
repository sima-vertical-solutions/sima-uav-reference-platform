# SiMa UAV reference platform: what it does, what it needs, where it goes

This document describes the features of the current release, the sensors that unlock each of
them, what a customer gains from starting here, and how the platform is meant to be extended.
For installing see `INSTALL.md` inside a release; for every tunable value see
[`CONFIGURATION.md`](CONFIGURATION.md).

## 1. What the platform is

A complete onboard software stack for an autonomous drone, running on a SiMa.ai Modalix board
under ROS 2. The board does the perception and odometry work on its machine learning
accelerator (MLA), at sensor rate, in the power budget of a small airframe, and hands the flight
controller what it needs to fly. The release is installable as-is on a flashed board, and every
part of it is configuration-driven so it can be adapted rather than rewritten.

The release is a reference, not a product: it shows one working, measured integration of
camera, radar and flight controller, and it is built so that each of those can be swapped.

Release 0.5.2 is the first for **platform 3.0**: eLxr 3.0 on the board, SiMa NEAT 0.6.0 as the
accelerator runtime, Python 3.13. Boards on eLxr 2.1.3 stay on the 0.4.x line; the README's
first table says which release a board takes.

## 2. Features in release 0.5.2

### Two airframes from one archive

The release carries a system config per drone, chosen at install with `deploy.sh --sys-config`.
It names the camera driver and with it the calibration, the mount, the odometry parameters and
the perception input size, and says whether a radar is fitted.

| | the Mistral (`mistral_config.yaml`) | the x650 (`x650_config.yaml`, the archive's default) |
|---|---|---|
| camera | MIPI, Sony IMX568 class, 1920x1080 at 30 fps | ELP global-shutter USB camera, 1280x960 at 30 fps |
| radar | TI IWR6843ISK, on | none fitted; `--radar` when one is |
| flight controller | CubeOrange+ over USB, MAVLink 2 at 921600 | the same |
| camera mount | measured in flight against the gyro, optical axis 46 degrees down | Kalibr calibration |

### Camera capture

One node owns the camera and publishes it in the NV12 format the accelerator consumes natively,
with the intrinsics alongside every frame. Nothing converts colour on the CPU for the MIPI
camera; the USB camera's MJPEG stream is decoded once and handed on the same way. The same frames
feed perception and odometry, so the two never contend for the device. When the camera stops
delivering, the publisher rebuilds its pipeline after 2 s without a frame and logs the gap, and
on the x650 a watchdog waits out that restart before acting.

### On-device segmentation

A YOLOv8 instance-segmentation model runs on the MLA at the camera's full rate, about 30 Hz,
producing class, bounding box and mask per detection. Thresholds, class limits and the model
archive itself are configuration.

### Radar and camera fusion

Detections from the camera are matched with points from the TI IWR6843ISK millimetre-wave
radar. Each fused detection carries a range and a radial velocity, gated by the segmentation mask
so the range belongs to the object and not its background. Output is at the radar's rate, 20 Hz,
up to the radar's physical limit of about 11 m. Every detection says whether radar contributed,
so a consumer can tell "no radar on this vehicle" from "radar saw nothing". The radar profile is
named in the system config (`sensors.radar.cfg`); unset, the driver's Standard profile is used.

### Visual-inertial odometry

The odometry is tightly coupled visual-inertial in this release. SuperPoint feature extraction
and LightGlue matching run on the MLA; the matches become feature tracks, and a GTSAM fixed-lag
smoother (iSAM2, a 6 s window) fuses them with the flight controller's IMU, preintegrated, and
its barometric height into an absolute pose at 30 Hz. The IMU and barometer come over the same
MAVLink link the pose goes out on. A rotor-drag term holds the velocity through frames the camera
sees nothing in, and each camera has its own IMU noise, clock-offset and drag values.

The pose is sent to the flight controller as a vision position estimate for GPS-denied flight.
**On hardware the link ships on**: both system configs name the odometry topic as the bridge's
`flight_controller.bridge.pose_topic`. Section 6 says what to check before trusting it and how
to switch it off. The pose is also published as a ROS odometry topic for bench checks and
recording.

### One MAVLink link to the flight controller

A single bridge node owns the connection to the flight controller: it receives the IMU,
attitude, barometer and EKF position, sends the odometry (the VIO pose, or a radar velocity),
carries target pursuit's GUIDED setpoints, and tunnels the ground-station command bridge's
packets. It is started whenever odometry or target pursuit runs, so no two stacks fight over the
UDP port. On the board a MAVLink relay (mavlink-router) multiplexes the controller's serial link:
the bridge listens on one of its outputs, telemetry on another.

### Flight-controller telemetry

A telemetry node asks the controller only for what the bridge does not: the GNSS fix, the EKF
status and variances, status text, the mission, home. It publishes them, and the EKF, GPS and
odometry tracks as paths in one local frame, so a flight can be watched in Foxglove with the
three position stories side by side.

### Target pursuit, opt-in

`deploy.sh --target-pursuit` adds a stack that follows one vehicle or person: a tracker on the
segmentation output (cars, trucks, people), re-identification through lighting changes with a
DINO model on the MLA, guidance that keeps a chosen track at a set distance and altitude, and a
local planner that sends GUIDED setpoints through the bridge. The target is chosen by a ROS
service or from QGroundControl over the telemetry radio, through the ground-station command
bridge. It is opt-in because it flies the drone; a plain launch never starts it. With
`--odometry radar --no-vo` the controller navigates on radar odometry's velocity instead of the
VIO pose.

### Object avoidance, opt-in

`--object-avoidance` adds the map the planner flies around: Depth Anything V2 on the MLA gives
relative depth from the camera, the radar scales it to metres and confirms the range, and a
local obstacle map in the map frame is built from both. Without it the planner flies the goal
straight.

### Recording

A bag recorder runs with the stack, writing zstd-compressed MCAP to external storage with a
free-space guard. The default profile records everything, frames included, so a flight can be
replayed through the odometry and the EKF, GPS and odometry tracks sit next to the pictures
that produced one of them; a throttled profile for long sorties keeps the frames at 5 Hz. The
recorder refuses to write to the board's eMMC, and a recording failure stops only the recording.

### Simulation with the same stack

The release ships the simulation side: Isaac Sim scenes of an urban environment in several
variants, a quadrotor model, an ArduPilot SITL bridge, and publishers that deliver frames, IMU,
rangefinder and radar data over ROS 2. The board runs exactly the same perception and odometry
code against the simulator as against the sensors. Only a handful of configuration values change,
and the install flag `--mode sim` applies them. This is how new scenes, flight patterns and model
variants are evaluated before they reach an airframe.

### Operations and verification

- **Health check before install.** `provision.sh` reports the flight controller, radar, camera
  (by capturing a frame, not just listing the device), the ROS 2 base, the NEAT version against
  the one the release was built with, the MAVLink relay and its telemetry port, and says what
  each gap costs.
- **Install, update, roll back.** `deploy.sh` ships the overlay, resolves the camera identity on
  the board, bakes the drone, the hardware declaration and the opt-in stacks into the run
  wrappers and verifies integrity with SHA-256 before anything is copied. Rolling back is
  deploying the previous folder.
- **Acceptance gate.** A rate checker measures every topic against the rates that were measured on
  the reference vehicle, with a profile per hardware configuration, so "it launched" and "it runs
  at speed" are different results. The same contract can run in CI on a board in the lab.
- **Operator view.** A terminal UI shows each stack's status and logs with restart and stop keys,
  starts the radar on a board installed with `--radar`, and on the x650 offers its square test
  mission. A Foxglove bridge with shipped layouts gives a graphical view of detections, radar and
  odometry.
- **Pipeline health check** watches the topic chain for staleness.
- **Traceability.** Every release carries a version stamp with the exact source, platform and
  model checksums it was built from, and the board reports what it runs.

### Board bring-up documentation

Step-by-step guides for radar firmware flashing, the MAVLink serial-to-UDP relay, a WiFi hotspot
on the board, an LTE modem and the Foxglove bridge, plus the x650's own board scripts. These are
the one-time steps a release does not automate.

## 3. Sensors and hardware

### The reference set

| component | reference part | interface | what it enables |
|---|---|---|---|
| compute | SiMa.ai Modalix DevKit on eLxr 3.0 with NEAT 0.6.0 | | everything; the MLA runs segmentation, the two odometry models, re-identification and depth |
| camera, Mistral | MIPI camera, Sony IMX568 class, 1920x1080 at 30 fps | MIPI CSI | segmentation, fusion, odometry |
| camera, x650 | ELP global-shutter USB camera, 1280x960 at 30 fps | USB (UVC, MJPEG) | the same |
| radar | TI IWR6843ISK, small-obstacle-detection firmware | USB (two serial ports) | range and velocity on detections; the metric scale of the depth map; radar odometry |
| flight controller | CubeOrange+, ArduPilot, EKF3 | USB or UART, MAVLink 2 at 921600 baud | the IMU and barometer the odometry fuses; the consumer of the vision pose and the pursuit setpoints |
| MAVLink relay | mavlink-router on the board | | fans the controller's serial link out to the bridge, telemetry and a ground station |
| telemetry radio | SiK radio to the ground station | | QGroundControl, and target selection from it |
| network | USB-C Ethernet, WiFi hotspot or LTE | | installing, operating and observing the board |

### What each sensor adds, and what its absence costs

Only the camera is a hard requirement; the installer refuses to run without one. Everything else
degrades gracefully and is declared at install time so the health check and acceptance gate judge
the vehicle you actually have.

| configuration | install flags | you get | you lose |
|---|---|---|---|
| camera + radar + flight controller | `--sys-config mistral_config.yaml` (Mistral) or `--radar` (x650) | the full stack: ranged detections, visual-inertial odometry, vision pose to the controller, the option of pursuit with avoidance | nothing |
| camera + flight controller | the x650 default, or `--no-radar` | segmentation at 30 Hz, visual-inertial odometry, pursuit | range and velocity on detections; the fusion output is silent; no obstacle map, no radar odometry |
| camera only | `--no-radar --no-fc` | segmentation; the odometry still runs, as a debug topic | the controller's IMU and barometer in the odometry, and any effect on flight |
| no sensors, replaying a recording | `--no-camera --no-radar` | the whole stack exercised from a bag | live operation |
| simulation | `--mode sim --sim-host <ip>` | perception, odometry and pursuit against Isaac Sim, with simulated IMU, rangefinder and radar | the real sensors' timing |

For optimal use, fit all three. The radar is what turns a segmentation mask into an obstacle at a
known distance and closing speed, and the flight controller is what turns a visual pose into
flight.

### Off-board

- A laptop with ssh for installing and for Foxglove.
- A ground station running QGroundControl over the telemetry radio, for arming and for target
  selection.
- For simulation, a workstation with an NVIDIA GPU running the Isaac Sim 6.1 container.

## 4. What a customer gains

- **Time to first result.** A flashed board becomes a perceiving, odometry-producing drone computer
  in one install with no build step. The health check tells you up front what the board is missing.
- **Perception on the edge, in the power budget.** Segmentation, feature extraction, matching,
  re-identification and depth all run on the MLA at sensor rate. No GPU, no ground link, nothing
  that stops working when the radio does.
- **GPS-denied flight path.** The visual-inertial pose reaches the flight controller through one
  bridge, with the ArduPilot parameter set beside it, so the remaining work is measurement and a
  bench check on your airframe rather than integration.
- **A second airframe already carried.** The x650 shows what adapting to another drone takes: a
  camera config with its calibration and mount, an odometry parameter layer, and a system config.
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

Thresholds, topics, rates, range gates, mount angles, intrinsics, the odometry's noise model, the
recording profile. All in the YAML files under the configuration tree, documented in
`CONFIGURATION.md`. On the board, edit the installed copy and restart to try a value; put it in
the client repository to keep it.

### Swap: a different model or sensor

- **Models.** Any NEAT MPK archive with a supported decode family can replace the segmentation
  model by changing `model_path` and `decode_type`; the inference node supports YOLOv8 detection,
  segmentation and pose, SSD and others. The two LightGlue archives shipped are selected the same
  way. Model identity is pinned by checksum in the manifest.
- **Camera.** A camera is a file in `config/camera/`: its driver, resolution, calibration and
  mount, plus an odometry parameter layer of the same name. The system config names it; the
  camera's identity is resolved on the board at install.
- **Radar.** A different radar profile is a different `.cfg`, named in the system config; the
  fusion node consumes any PointCloud2 with an optional Doppler field. A different radar is a
  different driver publishing that message.
- **Flight controller.** The bridge speaks MAVLink over UDP; any controller that accepts vision
  position estimates and GUIDED setpoints works behind the relay.
- **Airframe.** A system config: camera type, radar fitted or not, controller port, bridge and
  telemetry endpoints, recording profiles.

### Add: a new capability

A new stack is a launch include plus one dependency line in the bring-up package. Capabilities
live in the shared library and are selected by package name, so a client picks what it needs and
nothing else is built. Target pursuit and object avoidance were added this way, as opt-in stacks
on the same detections. The rover built on the same library uses RTAB-Map SLAM and Nav2 where
this drone uses visual-inertial odometry; both are selections, not forks.

### Simulate: a new scene or vehicle

Scenes are built from a base city with variant generators for lighting, obstacles and moving
people. A new airframe is a new USD asset and a motor model. Flight patterns are scripted and
produce reports, so a change is measured before it is flown.

## 6. Before you fly: vehicle-specific calibration

A reference platform cannot ship values that belong to one airframe. In 0.5.2 the odometry link
to the flight controller is **on by default on hardware**, with the values measured on the two
reference drones. A confidently wrong pose is worse for an EKF than none, so on any other
airframe either switch the link off until the values below are measured, or keep the aircraft
restrained until the bench check has passed.

**Switching the link off** is one line in the system config: `flight_controller.bridge.pose_topic: ""`.
The odometry still runs and publishes its ROS topic; nothing reaches the controller.

| value | where | what it is | how to get it |
|---|---|---|---|
| camera mount | `mount:` in `config/camera/<type>.yaml`: `cam_to_body_roll/pitch/yaw`, `cam_offset_m` | rotation and offset from the camera's optical frame to the airframe's body frame | from the mechanical mount, a Kalibr calibration, or a short flight comparing the camera's rotation against the gyro |
| camera clock offset | `vio.cam_time_offset_s` in `config/vo/params/<type>.yaml` | the camera's timestamps against the controller's IMU | from a Kalibr calibration, or the odometry's own offset monitor on a recorded flight |
| IMU noise | `vio.gyro_*`, `vio.accel_*` in the same file | the noise densities and random walks the smoother weights the IMU with | from the controller's IMU datasheet or an Allan-variance log |
| relay endpoints | `mavlink_connection` in `config/fc_bridge/flight_controller_bridge.hw.yaml`; `flight_controller.telemetry.connection` in the system config | the relay's outputs the bridge and telemetry listen on (reference: UDP 14600 and 14580 on the board) | from the mavlink-router configuration on the board; confirm with a heartbeat |

The `mavlink.*` keys and the height offset that `CONFIGURATION.md` section 7 lists belong to the
0.4.x odometry node, which spoke MAVLink itself; in 0.5.2 the bridge is the only MAVLink speaker
and the system config's `pose_topic` is the switch.

Two more values are per vehicle and already filled for the reference airframes. Re-measure them
for yours:

- **Camera intrinsics** (`fx fy cx cy` and distortion), in `config/camera/<type>.yaml`. A
  standard checkerboard or Kalibr calibration at the camera's native resolution.
- **Radar-to-camera extrinsic**, published as a static transform. Measured on the mount; the fusion
  node's `debug_projection` output exists to check it visually.

### The flight controller side

The board's half is not enough: ArduPilot has to be told to accept vision and to fly without GPS.
The release ships the parameter sets it uses in simulation under `sim/common/params/`;
`gps_denied_hitl_pose.parm` is the one for the VIO pose, and the same values apply on hardware
with the exceptions noted below.

| parameter | value | why |
|---|---|---|
| `VISO_TYPE` | `2` | vision position estimates (pose), as the bridge sends them. At 0 every vision message is silently dropped and arming is refused |
| `VISO_POS_X/Y/Z` | camera position relative to the controller, metres, body frame | per vehicle: measure on the airframe |
| `VISO_DELAY_MS` | `100` | the odometry's latency; must match the algorithm in use |
| `EK3_SRC1_POSXY` | `6` (ExternalNav) | the board sends an absolute pose in its own frame |
| `EK3_SRC1_VELXY` | `6` (ExternalNav) | EKF3 does not differentiate the pose, so the velocity is sent too |
| `EK3_SRC1_POSZ` / `EK3_SRC1_VELZ` | `1` (barometer) / `0` | vision-only height drifts without bound; every drone has a barometer |
| `EK3_SRC1_YAW` | `6` (ExternalNav) | heading from the pose |
| `GPS1_TYPE` | `0` | GPS-denied. Leave your GPS enabled if you want it as a fallback source instead |

For radar odometry (`--odometry radar`) the controller takes velocity deltas instead:
`radar_odom.parm`, with `VISO_TYPE 1`, `EK3_SRC1_POSXY 0`, `EK3_SRC1_YAW 1` (compass) and
`VISO_DELAY_MS 50`.

Do not copy `ARMING_SKIPCHK -1` or the `SIM_*` and `FRAME_*` lines onto a real aircraft; they are
for the simulator. Also set the controller's serial port and baud for the companion link to match
`flight_controller.serial` and `.baud` in the system config (reference: `/dev/ttyACM0`,
921600, MAVLink 2), and make sure the relay forwards the controller's messages to the bridge.
Without the IMU and barometer arriving, the smoother has nothing to fuse the features with.

### Other per-vehicle settings

- **Radar profile.** Named by `sensors.radar.cfg` in the system config; unset, the driver's
  Standard profile. The driver keeps the files in its installed share on the board, and a reflash
  resets them. The frame rate must stay at the 20 Hz the fusion and the acceptance gate expect, and
  `aoaFovCfg` should match the radar's field of view on your mount.
- **DDS domain.** Two vehicles on one network must have different `ROS_DOMAIN_ID`s, or they see
  each other's topics and the acceptance gate can pass on a neighbour's traffic. Set it with
  `deploy.sh --domain`, or use `--isolated` on a vehicle that talks only to itself.
- **Acceptance rates.** The expected topic rates in `acceptance.yaml` were measured with the
  reference cameras at 30 fps. A different camera changes them, and the gate will say so.
- **Recording storage.** `rosbag.recording_path` in the system config must be on external storage
  (`/media/bags` by reference); the recorder refuses the eMMC. The full-rate profile writes about
  5 GB a minute before compression.

### Bench check before the first flight

1. Run `provision.sh` without flags and get `board READY`.
2. Install, launch, and run the rate gate: every topic at its expected rate, the odometry at 30 Hz.
3. With the aircraft **restrained** and armed only from the ground station, move it by hand and
   watch the EKF's position in the ground station follow. Check that the direction matches: a
   wrong mount angle shows up here as motion on the wrong axis.
4. Check the vision message rate and the EKF's innovation on the ground station. Rejected vision
   data means the clock offset, the mount or the delay is wrong.
5. Only then fly, in a mode that holds position, low and short, without target pursuit.

Arm only from the ground control station, never from this software. Target pursuit is the one
stack that commands the aircraft, and it is never started unless asked for at install.

And one setting that is not calibration but is still yours to change: the WiFi hotspot's SSID
and passphrase in `provisioning/wifi/`. The release ships a default. Never fly on it.

## 7. Known limitations in this release

- 0.5.2 has not been flown, and the rate gate on a board did not run for this build in CI;
  earlier builds of the line passed it on a Modalix DevKit with the camera and no radar. The
  release notes say what was tested.
- The archive's `INSTALL.md` and `CONFIGURATION.md` carry text from the 0.4.x line: the NEAT
  install from `prereq/neat/` (0.5.2 ships no NEAT; the board's eLxr 3.0 provides 0.6.0), the
  odometry's `mavlink.*` keys and height offset (replaced by the bridge and the system config's
  `pose_topic`), and a run section that lists three stacks where 0.5.2 starts seven.
- The archive defaults to the x650; a Mistral is installed with `--sys-config mistral_config.yaml`.
- Values still marked unverified: the radar driver's Doppler field name, the Mistral camera's
  position offset, and target pursuit's camera latency.
- The vehicle description (URDF) does not exist yet; the TF tree is the camera and radar mounts.
- The alternative LightGlue archive is shipped and selectable by `model_path`, not loaded by default.
- Target pursuit and object avoidance are reference implementations exercised in simulation and on
  the bench; they are not a flight-tested autonomy product.
