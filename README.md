# vivado-docker

Builds a Docker image containing **AMD Vivado 2025.2 ML Standard** — the free
tier, good for most 7-series and UltraScale targets.

Works for **Apple Silicon** (M1/M2/M3/M4 via Rosetta) at least.

Fork of [filmil/vivado-docker](https://github.com/filmil/vivado-docker).

> **This repo contains no Vivado software.** You supply the installer; this
> repo supplies the recipe.

---

## Prerequisites

```bash
brew install --cask orbstack
```

| Requirement | Notes |
|-------------|-------|
| OrbStack | Provides Docker runtime on macOS — no separate Docker install needed |
| Vivado 2025.2 Unified Installer | Download from AMD — see step 1 |
| ~130 GB free disk during build | Peak usage; final image is ~30–40 GB (Artix-7 only) |

---

## Quick start

### 1 — Download the installer

Get the Vivado Unified Installer from
[amd.com/en/support/downloads](https://www.amd.com/en/support/downloads) (AMD
forces you to create an account just to download). The 2025.2 archive is named:

```
FPGAs_AdaptiveSoCs_Unified_SDI_2025.2_1114_2157.tar
```

Place it in the root of this repository.

### 2 — Choose your devices

Edit `install_config.txt`. The `Modules=` line controls which FPGA families are
installed. Enable only what you need — every extra family adds several GB.

Current defaults: **Artix-7**, **Spartan-7**, **Kintex-7**.

### 3 — Build

```bash
make HOST_TOOL_ARCHIVE_NAME=FPGAs_AdaptiveSoCs_Unified_SDI_2025.2_1114_2157.tar build
```

The Dockerfile uses `--mount=type=bind` so the 96 GB installer archive is never
copied into a Docker layer — halving peak disk usage vs. the traditional `COPY`
approach.

Typical durations:
- extract ~30 minutes,
- install ~30 minutes,
- layer export ~30 minutes.

### 4 — Run

```bash
# Interactive TCL console
VIVADO_CMD="vivado -mode tcl" ./run.vivado.sh

# Batch synthesis
SRC_DIR=/path/to/project \
WORK_DIR=/path/to/output \
VIVADO_CMD="vivado -mode batch -source /src/build.tcl" \
./run.vivado.sh
```

**`run.vivado.sh` environment variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `VIVADO_VERSION` | `2025.2` | Vivado version tag |
| `SRC_DIR` | `$PWD` | Host path mounted read-write at `/src` |
| `WORK_DIR` | `$PWD` | Host path mounted read-write at `/work` |
| `VIVADO_CMD` | `vivado` | Command executed inside the container |
| `ROSETTA` | auto-detect | Set `1` to force-enable the libudev stub |

---

## Note: Apple Silicon (Rosetta)

Vivado is x86\_64-only. On Apple Silicon, Docker runs the container under
Rosetta emulation. Two known crashes affect Vivado under Rosetta — both are
handled automatically by this fork.

**1. `realloc(): invalid pointer` on startup**

Vivado's license manager calls `udev_enumerate_scan_devices()`, which triggers
a heap corruption crash in glibc under Rosetta. This fork builds a minimal C
stub (`docker/udev_stub.c`) into the image at `/opt/udev_stub.so`. The stub
provides no-op implementations returning empty device lists. `run.vivado.sh`
detects ARM64 hosts and sets `LD_PRELOAD=/opt/udev_stub.so` automatically.

On native x86\_64 the stub is present but never loaded.

**2. `launch_runs` crashes in batch mode**

`launch_runs` spawns child processes that each re-load libudev independently
and crash, even with `LD_PRELOAD` set. Use in-process TCL commands instead:

```tcl
synth_design → opt_design → place_design → route_design → write_bitstream
```

---

## Credits

Based on [filmil/vivado-docker](https://github.com/filmil/vivado-docker) —
[BSD-style license](LICENSE).
