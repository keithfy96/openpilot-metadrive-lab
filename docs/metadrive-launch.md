# MetaDrive Launch Guide

Use this guide from the OpenPilot checkout:

```bash
cd /home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-metadrive-lab/openpilot
```

## Terminal 1: Start OpenPilot Manager

Run this and leave it running:

```bash
env PATH=/home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-metadrive-lab/openpilot/.venv/bin:$PATH \
  CI=1 \
  BLOCK=soundd \
  ./openpilot/tools/sim/launch_openpilot.sh
```

`BLOCK=soundd` avoids this host issue:

```text
OSError: PortAudio library not found
```

To fix sound support instead of blocking it:

```bash
sudo apt install -y libportaudio2
```

If the manager prints local bridge files followed by `needs update`, that is OpenPilot noticing local source edits in the simulator bridge. It is expected for this lab checkout after adding `--render` and window-key forwarding.

## Terminal 2: Start MetaDrive Bridge

Run this in a second terminal for a visual MetaDrive window:

```bash
cd /home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-metadrive-lab/openpilot
uv run --python 3.12.12 ./openpilot/tools/sim/run_bridge.py --render --control-window
```

For the original offscreen mode, omit `--render`.

## Controls

In visual mode, controls work from the separate `openpilot sim controls` window.

Keep the control window focused for keyboard input. The MetaDrive render window may accept mouse camera movement without accepting keyboard input on this desktop.

```text
1 = Cruise resume / accel
2 = Cruise set / decel / engage
3 = Cruise cancel
r = Reset simulation
i = Toggle ignition
q = Quit
wasd = Manual control
```

In `--render --control-window` mode, the ego vehicle mesh is visible and `wasd` in the control window sends manual driving commands while OpenPilot is not engaged.

This lab uses comma.ai's `metadrive` `minimal` branch, but that checkout did not include the default visible vehicle mesh files. The missing `ferra` vehicle assets were copied from the full upstream MetaDrive checkout into:

```text
openpilot-metadrive-lab/metadrive-comma/metadrive/assets/models/ferra/
```

Without those files, `--render` can fail during `env.reset()` with:

```text
OSError: Could not load model file(s): .../models/ferra/right_tire_front.gltf
```

Watch for:

```text
Ignition: True Engaged: False
```

After pressing `2`, the target state is:

```text
Ignition: True Engaged: True
```

## Render Modes

Visual mode:

```bash
uv run --python 3.12.12 ./openpilot/tools/sim/run_bridge.py --render --control-window
```

Expected log:

```text
Render Mode: onscreen
```

Offscreen mode:

```bash
uv run --python 3.12.12 ./openpilot/tools/sim/run_bridge.py
```

Expected log:

```text
Render Mode: offscreen
```

With `--render --control-window`, the bridge shows the ego vehicle and opens a small Tk control window. Manual `wasd` driving is sent through the control window while OpenPilot is not engaged. Without `--control-window`, controls are read from Terminal 2.

## Keyboard Debugging

If the MetaDrive window responds to mouse movement but not keyboard input, restart Terminal 2 with:

```bash
uv run --python 3.12.12 ./openpilot/tools/sim/run_bridge.py --render --control-window --input-debug
```

Click the `openpilot sim controls` window and press `w`, `a`, `s`, and `d`. The control window should show the last command it sent. If you also test the MetaDrive render window directly, useful debug lines look like:

```text
[sim-input] panda keyboard=True keys=['w']
[sim-input] window command: throttle_1.0
[sim-input] metadrive controller action: steer=0.000 gas=0.100
```

If `panda keyboard=False` or `keys=[]` continues while the window is focused and keys are pressed, Panda3D is not receiving keyboard events from the desktop session even though mouse events work. In that case, keep Terminal 2 focused and use terminal keyboard control, or use the later GUI-controller fallback.

Manual control messages are held briefly by the bridge so single key events are visible to the simulator instead of lasting only one 100 Hz tick.

If the bridge crashes with `AttributeError: 'NoneType' object has no attribute 'accept'`, update to the latest local patch in this lab checkout. Window key handlers must be registered after MetaDrive initializes its engine.

If the bridge is launched from a non-interactive command wrapper such as `timeout`, it may print:

```text
termios.error: (25, 'Inappropriate ioctl for device')
```

That comes from the terminal keyboard reader expecting a real TTY. Run the command directly in an interactive terminal for normal use.
