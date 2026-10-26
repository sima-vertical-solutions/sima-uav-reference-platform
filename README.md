# Mistral drone stack — releases

Built releases of the Mistral drone perception and visual-odometry stack for the SiMa.ai Modalix
DevKit. This repository holds **only the release archives**; the source is maintained privately.

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
