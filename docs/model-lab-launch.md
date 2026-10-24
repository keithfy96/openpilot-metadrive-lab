# OpenPilot Model Lab Launch Guide

Use this guide for the clean model-testing copy:

```bash
cd /home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-model-lab/openpilot
```

This lab is separate from the visual MetaDrive demo lab. It should stay close to upstream OpenPilot for model testing.

## Current State

- OpenPilot checkout: `openpilot-model-lab/openpilot`
- Python: `3.12.12`
- Virtualenv: `openpilot-model-lab/openpilot/.venv`
- Git status: clean against `origin/master`
- Model artifact present:

```text
openpilot/selfdrive/modeld/models/driving_tinygrad.pkl.chunkmanifest
```

## Terminal 1: Start OpenPilot Manager

Run this and leave it running:

```bash
cd /home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-model-lab/openpilot

env PATH=/home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-model-lab/openpilot/.venv/bin:$PATH \
  CI=1 \
  BLOCK=soundd \
  ./openpilot/tools/sim/launch_openpilot.sh
```

`BLOCK=soundd` avoids the host PortAudio issue:

```text
OSError: PortAudio library not found
```

To fix sound support instead of blocking it:

```bash
sudo apt install -y libportaudio2
```

## Terminal 2: Start Stock Simulator Bridge

Run this in a second terminal:

```bash
cd /home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-model-lab/openpilot

uv run --python 3.12.12 ./openpilot/tools/sim/run_bridge.py
```

This is the stock OpenPilot simulator bridge command. The clean model lab does not include the custom `--render`, `--control-window`, or `--input-debug` options from the visual demo lab.

## Controls

Terminal 2 must be focused for keyboard controls:

```text
1 = Cruise resume / accel
2 = Cruise set / decel / engage
3 = Cruise cancel
r = Reset simulation
i = Toggle ignition
q = Quit
wasd = Manual control
```

Watch for:

```text
Ignition: True Engaged: False
```

After pressing `2`, the target state is:

```text
Ignition: True Engaged: True
```

## Useful Verification Commands

Check that the model lab is clean:

```bash
cd /home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-model-lab/openpilot

env PATH=/home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-model-lab/openpilot/.venv/bin:$PATH \
  git status --short --branch
```

Expected:

```text
## master...origin/master
```

Check that the model manifest exists:

```bash
test -f openpilot/selfdrive/modeld/models/driving_tinygrad.pkl.chunkmanifest && echo model-manifest-present
```

Check Python:

```bash
.venv/bin/python --version
```

Expected:

```text
Python 3.12.12
```

## Notes

- Include `.venv/bin` on `PATH` for git commands in this checkout because OpenPilot's Git LFS hook expects `git-lfs`.
- `missing public key` in manager logs has appeared during simulator testing and is not the model-lab blocker by itself.
- Repeated `deleter hardwared` lines are manager status noise while the simulator stack is starting.
- Keep the visual demo changes in `openpilot-metadrive-lab`; keep model testing in `openpilot-model-lab`.
