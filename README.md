# SiMa UAV reference platform — releases

Installable releases of the **SiMa UAV reference platform**: the software that turns a SiMa
Modalix board into the onboard computer of an autonomous drone. Each release is one
self-contained archive with its installer, board health check, configuration and documentation,
verified by a SHA-256 checksum. Source is maintained privately; this repository holds only the
releases.

**Get the latest release from the [Releases page](https://github.com/sima-vertical-solutions/sima-uav-reference-platform/releases).**

What the current release does, which sensors it uses and how it is meant to be extended is in
[`PLATFORM_OVERVIEW.md`](PLATFORM_OVERVIEW.md). Every externalized parameter is listed in
[`CONFIGURATION.md`](CONFIGURATION.md).

## What a release contains

The current release is **0.4.2**. It attaches these files; later releases follow the same
naming with their own version number. The "Source code" archives GitHub adds automatically
contain only this README and the license and are not needed.

| asset | size | intended use |
|---|---|---|
| [`mistralsolutions-0.4.2.tar.gz`](https://github.com/sima-vertical-solutions/sima-uav-reference-platform/releases/download/0.4.2/mistralsolutions-0.4.2.tar.gz) | 349 MB | the release itself: everything below, in one folder |
| [`mistralsolutions-0.4.2.tar.gz.sha256`](https://github.com/sima-vertical-solutions/sima-uav-reference-platform/releases/download/0.4.2/mistralsolutions-0.4.2.tar.gz.sha256) | 96 B | checksum of the archive, checked before you unpack it |
| [`neat-stage-0.4.0-yolox-seg-pose.tar.gz`](https://github.com/sima-vertical-solutions/sima-uav-reference-platform/releases/download/0.4.2/neat-stage-0.4.0-yolox-seg-pose.tar.gz) | 113 MB | the exact SiMa NEAT runtime this release was built against (`0.4.0+feature-yolox-seg-pose`). The board needs it installed before `deploy.sh`; the release notes give the steps |
| [`neat-stage-0.4.0-yolox-seg-pose.tar.gz.sha256`](https://github.com/sima-vertical-solutions/sima-uav-reference-platform/releases/download/0.4.2/neat-stage-0.4.0-yolox-seg-pose.tar.gz.sha256) | 105 B | checksum of the NEAT archive |

Unpacking the archive gives a folder named `mistralsolutions-0.4.2/`:

```
mistralsolutions-0.4.2/
├── INSTALL.md
├── README.md
├── release_notes.txt
├── version.yaml
├── provision.sh
├── deploy.sh
├── check_rates.py
├── acceptance.yaml
├── install/
├── native_libs/
├── setup_overlay.sh
├── fastdds_large.xml
├── deploy.env
├── SHA256SUMS
├── provisioning/
└── sim/
```

Nothing in it needs a build machine, a container or a source checkout. **Do not edit anything inside it**: the installer
checks every file against `SHA256SUMS` and refuses to run if one has changed.

### Documents

| file | intended use |
|---|---|
| `INSTALL.md` | **start here.** The complete guide: what you need, first-time board bring-up, install, run, check, update, roll back, uninstall, troubleshooting, safety |
| `README.md` | one-page operator summary of the same folder, with the hardware the board needs and the flags that declare a peripheral absent |
| `CONFIGURATION.md` | every externalized parameter: which file it lives in, its hardware and simulation values, whether it has been verified on the target vehicle. Not inside the 0.4.2 archive; use [the copy in this repository](CONFIGURATION.md). Shipped in the archive from the next release |
| `release_notes.txt` | what changed since the previous release, and anything the board needs before installing |
| `version.yaml` | the identity of this build: release number, build time, source commits, base image, NEAT version and the checksum of every model. Copied to the board so it can report what it runs |

### Scripts you run

| file | intended use |
|---|---|
| `provision.sh` | board health check. Confirms the flight controller, radar, ROS 2 base, mavlink-router and the NEAT Python bindings are present, creates the log and capture directories, installs two small Python packages. Run it on a new or reflashed board, and after any cabling change. Flags `--no-radar`, `--no-fc`, `--mode sim` declare hardware the board is not meant to have |
| `deploy.sh` | the installer. Verifies `SHA256SUMS`, copies the software to `/root/sima_ws` on the board, writes the version stamp and generates the run scripts. Flags: `--board root@<ip>`, `--jump user@bastion`, `--mode hw|sim`, `--no-camera`, `--no-radar`, `--no-perception`, `--isolated`, `--run`, `--local` |
| `check_rates.py` | measures the publish rate of every topic in one window and passes or fails it against `acceptance.yaml`. The post-install "is it running at speed" check; `deploy.sh` installs a wrapper for it on the board |
| `acceptance.yaml` | the rate contract: the measured steady-state rate of each topic, how far it may fall, and the largest stall allowed, with one profile per board configuration (`hw`, `hw_no_radar`, `replay`, `sim`) |

### Installed on the board by `deploy.sh`

| file | intended use |
|---|---|
| `install/` | the built ROS 2 overlay, activated by `install/setup.bash`. Under `install/share/`: `camera_publisher`, `neat_inference` and `radar_camera_fusion` (perception), `visual_odometry` with `superpoint`, `lightglue` and `vo_motion` (odometry), `ti_mmwave_rospkg` (radar driver), `foxglove_bridge`, `system_monitor`, `system_config`, and `mistral_bringup` holding the launch files, this drone's configuration and the model archives `yolo_v8m_seg_mpk.tar.gz`, `superpoint_mpk.tar.gz`, `lightglue_mpk.tar.gz`, `lightglue_isaac_qat_mpk.tar.gz` |
| `native_libs/` | shared libraries the board's base image lacks in the right ABI, copied alongside the overlay. In 0.4.2: `libgtsam.so.4`, `libgtsam_unstable.so.4`, `libmetis-gtsam.so` and four `libboost_*.so.1.74.0` |
| `setup_overlay.sh` | sourced by the generated run scripts on the board to activate the overlay. Not run by hand |
| `fastdds_large.xml` | DDS transport profile that lets full-size camera frames pass over shared memory |
| `deploy.env` | the board wiring frozen at build time: client and bringup package names, remote directory, ROS domain ID, system config file. Read by the scripts, not edited |
| `SHA256SUMS` | integrity manifest of the whole folder; `deploy.sh` verifies it before shipping anything |

### Reference material, not installed

| folder | intended use |
|---|---|
| `provisioning/` | one-time board bring-up, done once per board before the first install. `provisioning/README.md` is the index |
| `provisioning/radar/SETUP.md` | flash the IWR6843ISK radar firmware (`small_obstacle_detection_68xx.bin`) with TI UniFlash |
| `provisioning/rpanion/SETUP.md` | install Rpanion-server and mavlink-router, which bridges the flight controller's serial link to UDP for the visual-odometry node |
| `provisioning/mavproxy/SETUP.md` | check the flight-controller link from a ground station with MAVProxy |
| `provisioning/wifi/SETUP.md`, `provisioning/wifi/Assets/` | WiFi hotspot: the 8852cu driver source (`wifi_file.zip`), `setup_hotspot.sh` and `hotspot_static.sh`, their systemd units, `NetworkManager.conf`, `rtw_options.conf`, `99-wifi-hotplug.sh`. **Set your own SSID and passphrase here** |
| `provisioning/lte/SETUP.md`, `provisioning/lte/Assets/` | LTE modem: `wwan_kernel_object.zip` and `wwan_setup.sh`, plus the APN setup |
| `provisioning/foxglove/` | Foxglove Studio layouts (`layouts/radar_fusion.hw.json`, `radar_fusion.sim.json`), the segmentation overlay extension `simaai.seg-overlay-0.0.8.foxe`, `coco_yolov8.json` class names, and the on-board launchers `run_bridge.sh` and `run_jpeg.sh` with `jpeg_republish.py` |
| `provisioning/secrets.example.env` | template for your WiFi interface, MAC, SSID, passphrase, AP address and LTE APN. Copy to `secrets.env`, fill in, keep out of the bundle |
| `sim/` | the simulator side of `--mode sim`. Runs on a GPU workstation with NVIDIA Isaac Sim, not on the board. `sim/README.md` is the guide, `sim/RELEASE_NOTES.md` what differs from its upstream |
| `sim/common/` | shared by every scene: `scripts/launch_sim.py`, `fly.py`, `build_quadrotor.py`, `check_sitl.py`, `check_frames.py`, `probe_camera.py`, `drift_report.py`; the SITL/HITL bridge under `src/`; `board/feature_bridge_isaac.yaml` for the board's sim camera |
| `sim/common/params/` | ArduPilot parameter sets, one per position source: `gps_sitl.parm`, `gps_denied.parm`, `gps_denied_hitl.parm`, `gps_denied_hitl_pose.parm`, `baro_only.parm`. `gps_denied_hitl.parm` is the reference for the vision-position settings in `PLATFORM_OVERVIEW.md` section 6 |
| `sim/urban-scene/` | GPS-denied flight in NVIDIA's Rivermark city: `launch_sim.sh`, `fly.sh`, `configs/sim*.yaml`, `docs/HITL.md`, scene and route scripts |
| `sim/school-scene/` | a second scene with the same layout: `launch_sim.sh`, `fly.sh`, `configs/sim.yaml`, `scripts/build_scene.py`, `survey_area.py` |
| `sim/report_flight.py` | scores a flown or simulated flight from its log |

## Installing, step by step

The full text with expected output and every variant is `INSTALL.md` inside the archive. The
sequence is the same for every release. Commands run on a Linux or macOS laptop with
passwordless `root` ssh to the board, unless noted.

1. **Download and verify.** For 0.4.2:

   ```bash
   curl -LO https://github.com/sima-vertical-solutions/sima-uav-reference-platform/releases/download/0.4.2/mistralsolutions-0.4.2.tar.gz
   curl -LO https://github.com/sima-vertical-solutions/sima-uav-reference-platform/releases/download/0.4.2/mistralsolutions-0.4.2.tar.gz.sha256
   sha256sum -c mistralsolutions-0.4.2.tar.gz.sha256
   tar -xzf mistralsolutions-0.4.2.tar.gz
   cd mistralsolutions-0.4.2
   ```

2. **Read `release_notes.txt`.** It says whether the board needs something before this release.
   0.4.2 needs NEAT `0.4.0+feature-yolox-seg-pose`, attached as
   `neat-stage-0.4.0-yolox-seg-pose.tar.gz`. Copy it and its `.sha256` to the board and, as root
   on the board:

   ```bash
   sha256sum -c neat-stage-0.4.0-yolox-seg-pose.tar.gz.sha256
   tar xzf neat-stage-0.4.0-yolox-seg-pose.tar.gz && cd neat-stage
   bash install_neat_framework.sh --local
   sha256sum /lib/firmware/modalix-cvu-fw                    # these two hashes must match;
   cat /usr/share/sima-neat-firmware/modalix-cvu-fw.sha256   # if not, run the line below
   /usr/libexec/sima-neat-firmware/activate-ev74-firmware.sh
   ```

3. **Check that you can reach the board.**

   ```bash
   ssh root@<board-ip> uname -m      # should print: aarch64
   ```

   If the board is only reachable through another machine, add `--jump user@bastion` to every
   script call below.

4. **First time on this board only: one-time bring-up.** Follow `provisioning/README.md`:
   flash the radar firmware, install mavlink-router for the flight-controller link, and set up
   WiFi or LTE. Put your credentials in a copy of `provisioning/secrets.example.env` and
   **set your own hotspot SSID and passphrase** (see "Before you fly" below). Skip this step if
   the board already runs an earlier release.

5. **Health check.**

   ```bash
   ./provision.sh --board root@<board-ip>
   ```

   It prints one line per item and ends with `RESULT: board READY for deploy.sh` or a list of
   gaps. `MISSING` on the flight controller or radar is cabling or step 4; it cannot be fixed by
   the install. If the board deliberately lacks a peripheral, say so, and the check stops
   counting it against you:

   ```bash
   ./provision.sh --board root@<board-ip> --no-radar            # camera, no radar
   ./provision.sh --board root@<board-ip> --no-radar --no-fc    # camera only
   ```

6. **Install.**

   ```bash
   ./deploy.sh --board root@<board-ip>
   ```

   About a minute over USB-C. The previous software under `/root/sima_ws` is replaced, nothing
   outside it is touched. Use the same `--no-radar` or other flags you gave the health check;
   `deploy.sh` bakes them into the run scripts so nobody has to remember them later. For a
   board fed by a simulator instead of sensors, use `--mode sim --sim-host <sim-ip>`.

   To install from the board itself, copy the folder onto it and run `./provision.sh --local`
   and `./deploy.sh --local` as root, keeping the folder outside `/root/sima_ws`.

7. **Run.**

   ```bash
   ssh -t root@<board-ip> /root/sima_ws/run_mistralsolutions_tui.sh    # operator TUI
   ssh    root@<board-ip> /root/sima_ws/run_mistralsolutions_stack.sh  # headless
   ```

   `Ctrl-C` stops everything, including the nodes it started. Add `--foxglove` to the headless
   command to bring the Foxglove bridge up with the stack, then point Foxglove Studio at
   `ws://<board-ip>:8765` and load a layout from `provisioning/foxglove/layouts/`.

8. **Check it is running at speed.** With the stack up, in a second shell:

   ```bash
   ssh root@<board-ip> /root/sima_ws/check_mistralsolutions_rates.sh
   ```

   It measures every topic for one window and reports PASS or FAIL against `acceptance.yaml`.

9. **Confirm what is installed.**

   ```bash
   ssh root@<board-ip> cat /root/sima_ws/version.yaml   # release: 0.4.2
   ```

**Updating** is step 1 and step 6 with the new folder. **Rolling back** is step 6 from an older
folder. **Uninstalling** removes `/root/sima_ws` on the board; nothing else was installed.

## Before you fly

A release may ship the board's WiFi hotspot with a **default SSID and passphrase** in its
`provisioning/wifi/` scripts and docs. That network gives root ssh access to the board to anyone
in WiFi range. Set your own before running the WiFi bring-up, and never fly a vehicle that still
uses the default. Each release's `INSTALL.md` says where.

The stack feeds visual-odometry pose to the flight controller. The values that depend on your
airframe (camera mount, height offset, MAVLink endpoint, flight controller parameters) must be
set for your vehicle before that link is enabled; `PLATFORM_OVERVIEW.md` section 6 lists them
and the bench check to run. Arm only from the ground station, with the aircraft restrained, until
the pose stream has been checked.

## Configuration

Every release is configured in layers, later ones winning: node defaults compiled into the
software, per-stack YAML files shipped in the archive, a hardware or simulation half picked by the
`mode` flag, values resolved on the board at install time, launch arguments, and finally the deploy
flags. Model archives are selected by a `model_path` parameter, so swapping a model is a config
change, not a rebuild. [`CONFIGURATION.md`](CONFIGURATION.md) lists every parameter.
