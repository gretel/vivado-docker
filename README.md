# vivado-docker (gretel fork)

Fork of [filmil/vivado-docker](https://github.com/filmil/vivado-docker) adapted
for **Vivado 2025.2** on **Apple Silicon** (Rosetta x86_64 emulation via
OrbStack or Docker Desktop).

See the upstream repo for general background, prior art, and contribution info.

## Why this fork?

Vivado is x86_64-only. Running it in Docker on Apple Silicon (M1/M2/M3/M4)
requires Rosetta emulation, which exposes two problems that upstream doesn't
address:

1. **libudev crash** -- Vivado's license manager and WebTalk telemetry call
   `udev_enumerate_scan_devices()`, which triggers a `realloc(): invalid
   pointer` crash in glibc's allocator under Rosetta. Our fork builds a tiny
   C stub (`docker/udev_stub.c`) into the image at `/opt/udev_stub.so` that
   provides no-op implementations returning empty device lists. The run script
   auto-detects ARM64 hosts and sets `LD_PRELOAD` so Vivado never sees the
   real libudev.

2. **`launch_runs` child-process crashes** -- `launch_runs` spawns child
   processes that each independently load libudev and crash even with
   `LD_PRELOAD`. The workaround is to use in-process TCL commands instead:
   `synth_design`, `opt_design`, `place_design`, `route_design`,
   `write_bitstream`.

Additional improvements over upstream:

- **`--mount=type=bind`** in the Dockerfile avoids duplicating the ~96 GB
  installer archive into a Docker layer, cutting peak disk usage roughly in
  half (~130 GB vs ~290 GB).
- **Single-layer `apt-get install`** with `--no-install-recommends` for a
  smaller base image.
- **`XILINX_LOCAL_USER_DATA=no`** disables WebTalk telemetry (the
  `config_webtalk` TCL command was removed in Vivado 2025.x).
- **Configurable `run.vivado.sh`** with env-var overrides for source dir, work
  dir, Vivado command, and Rosetta mode.

## Quick start

### 1. Download the installer

Get `FPGAs_AdaptiveSoCs_Unified_SDI_2025.2_1114_2157.tar` from AMD and place
it in this directory.

### 2. Edit `install_config.txt`

Enable only the device families you need. Fewer families = smaller image.
Current defaults: Artix-7, Spartan-7, Kintex-7.

### 3. Build

```bash
make HOST_TOOL_ARCHIVE_NAME=FPGAs_AdaptiveSoCs_Unified_SDI_2025.2_1114_2157.tar build
```

Takes ~90 min total (extract ~30 min, install ~30 min, layer export ~30 min).

### 4. Run

```bash
make run                                       # Vivado GUI
VIVADO_CMD="vivado -mode tcl" ./run.vivado.sh  # interactive TCL
SRC_DIR=/path/to/project WORK_DIR=/path/to/output \
  VIVADO_CMD="vivado -mode batch -source /src/build.tcl" \
  ./run.vivado.sh                              # batch synthesis
```

### `run.vivado.sh` environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `VIVADO_VERSION` | `2025.2` | Vivado version |
| `SRC_DIR` | `$PWD` | Host directory mounted at `/src` |
| `WORK_DIR` | `$PWD` | Host directory mounted at `/work` |
| `VIVADO_CMD` | `vivado` | Command to run inside container |
| `ROSETTA` | auto-detect | `1` to force libudev stub on ARM64 |

### Save / load image

```bash
make save                                     # export to .tgz
docker load -i xilinx-vivado.docker.tgz       # import on another machine
```

## Troubleshooting

**Vivado crashes with `realloc(): invalid pointer` on Apple Silicon**

The libudev stub should handle this automatically. If running Vivado outside of
`run.vivado.sh`, set `LD_PRELOAD=/opt/udev_stub.so` before sourcing
`settings64.sh`.

**`launch_runs` crashes but `synth_design` works**

`launch_runs` spawns children that crash under Rosetta even with `LD_PRELOAD`.
Use in-process TCL: `synth_design`, `opt_design`, `place_design`,
`route_design`, `write_bitstream`.

## License

See [LICENSE](LICENSE). Original work copyright 2023 Google (BSD-style).
