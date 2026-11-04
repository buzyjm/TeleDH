# TeleDH: deploying on a new machine

[README.md](README.md) covers what the software does and how to operate it.
This file covers what you need to get it running somewhere other than the
original lab machine.

```bash
git clone https://github.com/zhanglab-physicalAI/TeleDH
cd TeleDH
```

## What is and is not in the repository

| Included | Not included |
|---|---|
| `telehand/` package, `run.py`, `scripts/`, `tests/` | `vendor/`: Python packages built for one machine (Linux x86_64, CPython 3.12). Install your own, see below |
| `docs/` (architecture, experiments, frames, tactile, DH116 user manual PDF) | `.telehand/`: per-machine runtime state (saved camera, view angle, mesh cache, session lock) |
| `models/hand_landmarker.task` (MediaPipe model) | per-operator calibration (`calibration/handprofile*.json`, `direct_calibration.json`). Each operator makes their own with `run.py profile` |
| `calibration/poses.json` (pose library for the DH116 unit) | generated benchmarks, logs and screenshots under `data/` |
| `third_party/DH116_URDF_MJCF_Files` (robot model) | |
| `third_party/LHandProLib-API-Linux-20260727` (vendor hand SDK: x86_64, aarch64, i386) | |

## Requirements

- **Python 3.12 or newer.** The `vendor/` folder on the original machine was built for 3.12.
- **Linux, to drive the hand.** Hand control goes through the vendor's
  LHandProLib SDK, which is shipped only as Linux shared libraries
  (`libLHandProLib.so`). Without the hand, tracking and visualisation still work
  on other platforms.
- A webcam, a DH116 / LHandPro `LAC_DOF_6` hand on RS485 (USB-serial adapter),
  and a display for the OpenCV window.

## Install

The original machine had no `pip` or `venv`, so its README installs into
`vendor/` with pip's zipapp. On a normal machine use a virtual environment:

```bash
python3.12 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt     # pinned, tested versions
# or: pip install -e ".[render,dev]"  # looser ranges from pyproject.toml
```

`run.py` puts `vendor/` on `sys.path` if the folder exists and carries on
normally if it does not, so packages from a venv work unchanged.

Then:

```bash
python run.py check                    # camera, tracking, hand link; never moves the hand
python run.py cameras                  # list cameras; add --use <name> to remember one
python run.py profile --name <you>     # per-operator finger scaling, no motors
python run.py teleop --show-3d-pose --smoothing 0.7 --deadband 0.4 --max-speed 300 --no-serial-flush
```

Use the interpreter you installed into. On the original machine bare `python3`
pointed at a different Python (miniforge 3.14) than the packages had been built
for, and imports failed until it was run as `/usr/bin/python3.12`.

## Changes needed per environment

These values are hard-coded for the original machine. Change them before
deploying on different hardware or another OS.

| What | Where | Original value | Notes |
|---|---|---|---|
| Hand SDK architecture | `telehand/paths.py`, `SDK_ROOT` | `.../LHandProLib-API-Linux-20260727/x86_64` | Change to `aarch64` for ARM Linux (Jetson, Raspberry Pi) or `i386`. No Windows or macOS build is included |
| Serial port | `telehand/hand.py`, `DEFAULT_PORT`; or `--port` | `/dev/ttyUSB0` | Run `ls /dev/ttyUSB* /dev/ttyACM*` to find the adapter. The user needs access to the port (usually the `dialout` group: `sudo usermod -aG dialout $USER`, then log in again) |
| Camera backend | `telehand/camera.py`, `cv2.VideoCapture(target, cv2.CAP_V4L2)` | V4L2 | Linux-only. Camera discovery also reads `/dev/v4l/by-path` and `/sys/class/video4linux`. Other platforms need a different backend (e.g. `CAP_DSHOW`/`CAP_MSMF` on Windows, `CAP_AVFOUNDATION` on macOS) and a different way to list cameras |
| Camera choice | `--camera`, or `run.py cameras --use <name>` | saved per machine in `.telehand/` | Accepts an index, a `/dev/videoN` node, a by-path link or part of the camera's name. Index numbers change when cameras are replugged, so names are safer |
| Display | environment | `DISPLAY=:1` over SSH | Any X display works. Use `--no-display` to run headless |
| Baud rate | `--baud` | 500000 | Must match the hand's firmware setting |

## Verifying an install

```bash
python run.py --help
python -m pytest            # needs pytest (the "dev" extra); no hardware required
```

## Further reading

- [README.md](README.md): operating the hand, retargeting modes, tuning, latency profiling
- [docs/architecture.md](docs/architecture.md): code layout and data flow
- [docs/experiments.md](docs/experiments.md): measurements behind the default settings
- [docs/frames.md](docs/frames.md), [docs/tactile.md](docs/tactile.md): coordinate frames, tactile sensing
- `third_party/LHandProLib-API-Linux-20260727/*/docs/`: vendor SDK manual
