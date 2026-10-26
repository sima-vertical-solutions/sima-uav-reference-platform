# SiMa UAV reference platform — releases

This repository distributes releases of the **SiMa UAV reference platform**: installable software
stacks that turn an edge AI compute board into the onboard computer of an autonomous drone. Each
release is a complete, working example of the platform, from the board's base image to the
applications running on it, built to be installed as-is and then adapted.

The platform is designed to grow. Releases will cover different sensor sets, autonomy
capabilities and airframes, so no single release defines it. Every one of them is a
self-contained archive with its installer, board health check and configuration, verified by a
SHA-256 checksum and published automatically from the development repository. Source is maintained
privately; this repository holds only the releases.

[`PLATFORM_OVERVIEW.md`](PLATFORM_OVERVIEW.md) describes what the current release does, which
sensors unlock each feature, what a customer gains from it and how it is meant to be extended.

## Download

Each entry on the [Releases page](../../releases) ships an archive and its checksum:

| file | what |
|------|------|
| `<name>-<version>.tar.gz` | the self-contained release folder |
| `<name>-<version>.tar.gz.sha256` | its checksum |

```bash
curl -LO <release asset URL>
curl -LO <release asset URL>.sha256
sha256sum -c <name>-<version>.tar.gz.sha256
tar -xzf <name>-<version>.tar.gz
```

Then follow `INSTALL.md` inside the extracted folder. It lists the hardware that release targets,
which peripherals are optional, and the flags that declare one absent. The "Source code" archives
GitHub attaches to every release contain only this README and the license.

## Before you fly

A release may ship the board's WiFi hotspot with a **default SSID and passphrase** in its
`provisioning/wifi/` scripts and docs. That network gives root ssh access to the board to anyone
in WiFi range. Set your own before running the WiFi bring-up, and never fly a vehicle that still
uses the default. Each release's `INSTALL.md` says where.

## Configuration

Every release is configured in layers, later ones winning: node defaults compiled into the
software, per-stack YAML files shipped in the archive, a hardware or simulation half picked by the
`mode` flag, values resolved on the board at install time, launch arguments, and finally the deploy
flags. Model archives are provisioned separately and selected by a `model_path` parameter, so
swapping a model is a config change, not a rebuild.

[`CONFIGURATION.md`](CONFIGURATION.md) lists every externalized parameter of the current stack:
the file it lives in, its hardware and simulation values where they differ, and whether it has
been verified on the target vehicle. Its last section collects every value still marked as
unverified. From the next release the same file ships inside the archive beside `INSTALL.md`.

## Installing

Every release follows the same three steps, run from a laptop with ssh to the board or from the
board itself:

```bash
./provision.sh --board root@<ip> [flags]   # health check: is this board ready for this release?
./deploy.sh    --board root@<ip> [flags]   # install
./deploy.sh    --board root@<ip> --run     # install and launch
```

Peripherals a release treats as optional are declared with flags such as `--no-radar`, so the
health check does not report gaps for hardware the board is not meant to have. Run without flags
on a fully equipped vehicle, where a missing sensor is a fault.
