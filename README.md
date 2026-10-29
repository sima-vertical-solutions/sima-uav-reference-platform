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

Each entry on the Releases page attaches these files. The "Source code" archives GitHub adds
automatically contain only this README and the license and are not needed.

| asset | intended use |
|---|---|
| `<name>-<version>.tar.gz` | the release itself: everything below, in one folder |
| `<name>-<version>.tar.gz.sha256` | checksum of the archive, checked before you unpack it |
| `neat-stage-<version>.tar.gz` (+ `.sha256`) | the exact SiMa NEAT runtime this release was built against, when the release notes say the board needs it installed first |

Unpacking the archive gives a folder named `<name>-<version>/`. Nothing in it needs a build
machine, a container or a source checkout. **Do not edit anything inside it**: the installer
checks every file against `SHA256SUMS` and refuses to run if one has changed.

### Documents

| file | intended use |
|---|---|
| `INSTALL.md` | **start here.** The complete guide: what you need, first-time board bring-up, install, run, check, update, roll back, uninstall, troubleshooting, safety |
| `README.md` | one-page operator summary of the same folder, with the hardware the board needs and the flags that declare a peripheral absent |
| `CONFIGURATION.md` | every externalized parameter: which file it lives in, its hardware and simulation values, whether it has been verified on the target vehicle. (Shipped in the archive from the release after 0.4.2; for 0.4.2 use the copy in this repository) |
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
| `install/` | the built ROS 2 overlay: camera publisher, perception (segmentation on the MLA and radar-camera fusion), visual odometry, radar driver, Foxglove bridge, system monitor, launch files, this drone's configuration and the model archives under `share/` |
| `native_libs/` | shared libraries the board's base image lacks in the right ABI. Copied alongside the overlay; may be empty |
| `setup_overlay.sh` | sourced by the generated run scripts on the board to activate the overlay. Not run by hand |
| `fastdds_large.xml` | DDS transport profile that lets full-size camera frames pass over shared memory |
| `deploy.env` | the board wiring frozen at build time: client and bringup package names, remote directory, ROS domain ID, system config file. Read by the scripts, not edited |
| `SHA256SUMS` | integrity manifest of the whole folder; `deploy.sh` verifies it before shipping anything |

### Reference material, not installed

| folder | intended use |
|---|---|
| `provisioning/` | one-time board bring-up, done once per board before the first install. `README.md` is the index. `radar/` flashes the radar firmware; `rpanion/` installs mavlink-router, which bridges the flight controller to the visual-odometry node; `mavproxy/` checks that link from a ground station; `wifi/` and `lte/` set up the network you reach the board on; `foxglove/` holds Foxglove Studio layouts and the segmentation overlay extension; `secrets.example.env` is the template for your WiFi, APN and other credentials |
| `sim/` | the simulator side of `--mode sim`: NVIDIA Isaac Sim scenes (`urban-scene/`, `school-scene/`), ArduPilot parameter sets for GPS-denied flight under `common/params/`, the SITL/HITL bridge, flight scripts and `report_flight.py` for scoring a flight. Runs on a GPU workstation, not on the board |

## Installing, step by step

The full text with expected output and every variant is `INSTALL.md` inside the archive. The
sequence is the same for every release. Commands run on a Linux or macOS laptop with
passwordless `root` ssh to the board, unless noted.

1. **Download and verify.** From the Releases page, download the archive and its checksum, then:

   ```bash
   sha256sum -c <name>-<version>.tar.gz.sha256
   tar -xzf <name>-<version>.tar.gz
   cd <name>-<version>
   ```

2. **Read `release_notes.txt`.** It says whether the board needs something before this release,
   such as a specific NEAT runtime. If a `neat-stage-*.tar.gz` is attached, copy it to the
   board and install it as root, following the notes, before going on.

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
   ssh root@<board-ip> cat /root/sima_ws/version.yaml   # release: <version>
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
