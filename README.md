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
