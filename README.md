# SiMa.ai drone reference platform — releases

This repository distributes releases of the SiMa.ai **drone reference platform**: a complete,
installable software stack that turns a Modalix DevKit into the perception and odometry computer
of an autonomous drone. It pairs radar and camera fusion for obstacle detection with monocular
visual odometry that feeds pose to an ArduPilot flight controller over MAVLink, all running on the
SiMa.ai MLA under ROS 2 Jazzy.

The platform is built to be adopted and adapted. It ships with a reference sensor set, a TI
IWR6843ISK radar, a MIPI camera and a CubeOrange+ flight controller, but every peripheral beyond
the camera is optional and declared at install time, so the same release runs on a bench DevKit, a
camera-only airframe or a full drone. Each release is a self-contained archive with its installer,
board health check, configuration and quantized models, verified by a SHA-256 checksum and
published automatically from the development repository. Source is maintained privately; this
repository holds only the releases.

## Download

Each release on the [Releases page](../../releases) ships two files:

| file | what |
|------|------|
| `mistralsolutions-<version>.tar.gz` | the self-contained release folder |
| `mistralsolutions-<version>.tar.gz.sha256` | its checksum |

```bash
V=0.4.2
curl -LO https://github.com/sima-vertical-solutions/mistralsolutions-releases/releases/download/$V/mistralsolutions-$V.tar.gz
curl -LO https://github.com/sima-vertical-solutions/mistralsolutions-releases/releases/download/$V/mistralsolutions-$V.tar.gz.sha256
sha256sum -c mistralsolutions-$V.tar.gz.sha256
tar -xzf mistralsolutions-$V.tar.gz
```

Then follow `INSTALL.md` inside the extracted folder. The "Source code" archives GitHub attaches to
every release contain only this README and the license.

## What you need

| | required? | notes |
|---|---|---|
| SiMa.ai Modalix DevKit, flashed base image | yes | ROS 2 and a NEAT version matching the release (`provision.sh` checks it) |
| MIPI camera | yes, unless `--no-camera` | the installer refuses to install without one; `--no-camera` is for replaying a recording |
| TI IWR6843ISK radar | no | `--no-radar`: everything runs, detections carry no range |
| CubeOrange+ flight controller | no | without it, visual odometry runs but its pose reaches nothing |
| ssh to the board | yes | any network: USB-C Ethernet, WiFi or LTE |

Supported board configurations:

```bash
./provision.sh --board root@<ip>                        # camera + radar + flight controller
./provision.sh --board root@<ip> --no-radar             # camera only
./provision.sh --board root@<ip> --no-radar --no-fc     # camera only, no flight controller
./deploy.sh    --board root@<ip> --no-radar             # install matching the health check above
./deploy.sh    --board root@<ip> --mode sim             # simulation: no camera or radar on the board
```

Without the matching flags, `provision.sh` reports `board has GAPS` for hardware the board is not
meant to have. The flags are deliberate: on a fully equipped drone an unplugged sensor is a fault.
