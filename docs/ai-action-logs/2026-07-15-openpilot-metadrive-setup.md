# AI Action Log: OpenPilot + MetaDrive Setup

Date: 2026-07-15 16:23:16 +08
Workspace: `/home/keith/Desktop/work/wingfin/AI-orchestration`

## User Goal

Set up a minimal local test environment for OpenPilot with MetaDrive before building the full application.

## Actions Taken

- Created sandbox directory: `openpilot-metadrive-lab/`.
- Cloned OpenPilot into `openpilot-metadrive-lab/openpilot` with submodules.
- Cloned upstream MetaDrive into `openpilot-metadrive-lab/metadrive`.
- Created and populated OpenPilot virtual environment with `uv`.
- Used Python `3.12.12` because OpenPilot pins `3.12.13`, but that exact managed uv build was unavailable; `3.12.12` still satisfies OpenPilot's declared `>=3.12.3,<3.13` range.
- Installed upstream MetaDrive as an editable package into OpenPilot's `.venv`.
- Pulled MetaDrive assets successfully.
- Ran MetaDrive offscreen/headless verification successfully.
- Ran MetaDrive profile test for 10,000 simulation steps at about 939 FPS; it completed the simulation loop but exited with code `139` during teardown.
- Built targeted OpenPilot native artifacts needed by the simulator path after the full build failed on font asset generation.
- Launched OpenPilot simulator manager in `CI=1` mode; it eventually stayed running without immediate manager startup failure.
- Launched OpenPilot's MetaDrive bridge; it reached MetaDrive startup but failed on a config mismatch.
- Stopped the bridge and manager sessions; no OpenPilot or MetaDrive processes were left running.

## Files And Directories Created

- `openpilot-metadrive-lab/`
- `openpilot-metadrive-lab/openpilot/`
- `openpilot-metadrive-lab/openpilot/.venv/`
- `openpilot-metadrive-lab/metadrive/`
- `openpilot-metadrive-lab/metadrive/metadrive/assets/`
- `openpilot-metadrive-lab/metadrive/metadrive/examples/main_camera_from_observation.png`
- `openpilot-metadrive-lab/metadrive/metadrive/examples/main_camera_from_buffer.png`
- OpenPilot build outputs inside `openpilot-metadrive-lab/openpilot/`, including native `.so` files, generated MPC code, `.sconsign.dblite`, and `compile_commands.json`.
- OpenPilot boot logs under `~/.comma/media/0/realdata/boot`.
- uv cache/runtime files under `~/.cache/uv` and `~/.local/share/uv`.

## Important Artifacts Built

- `openpilot/common/params_pyx.so`
- `msgq_repo/msgq/visionipc/visionipc_pyx.so`
- `rednose_repo/rednose/helpers/ekf_sym_pyx.so`
- `openpilot/selfdrive/controls/lib/longitudinal_mpc_lib/c_generated_code/acados_ocp_solver_pyx.so`
- `openpilot/system/loggerd/bootlog`

## What Went Wrong

The main blocker was a MetaDrive version/fork mismatch.

OpenPilot's simulator bridge expects comma.ai's MetaDrive fork/branch, not the upstream `metadriverse/metadrive` package that was installed first. The bridge failed while spawning MetaDrive with:

```text
KeyError: "{'anisotropic_filtering'} does not exist in existing config"
```

This means OpenPilot passed a config key that exists in the MetaDrive version it expects, but not in the upstream MetaDrive checkout.

Other issues encountered:

- Initial network operations required approval because the sandbox has restricted network access.
- `uv sync` needed approval because it writes to `~/.cache/uv`.
- OpenPilot's exact `.python-version` of `3.12.13` was not available through the local uv managed Python index.
- Full `scons` failed on OpenPilot raylib/font asset generation, so targeted native modules were built instead.
- `nvidia-smi` could not communicate with the NVIDIA driver from this shell, while NVIDIA kernel modules were loaded; display/GPU should be verified from the actual desktop session.

## Current State

- OpenPilot checkout exists and has the required targeted native artifacts for the simulator path.
- Upstream MetaDrive was replaced with comma.ai's `metadrive` fork on the `minimal` branch in `openpilot-metadrive-lab/metadrive-comma`.
- The active Python environment now imports MetaDrive from `metadrive-comma`, version `0.4.2.3`.
- MetaDrive comma fork assets were downloaded successfully.
- OpenPilot Git LFS was initialized locally using `.venv/bin/git-lfs`, and LFS model/sound assets were checked out.
- The OpenPilot bridge CLI works with `-h`.
- The OpenPilot + MetaDrive bridge now starts and spawns MetaDrive offscreen through VisionIPC.
- OpenPilot did not engage because manager processes were unhealthy: `modeld`, `soundd`, `locationd`, and later `paramsd` showed failures before LFS and generated artifacts were fixed.
- `soundd` was fixed by Git LFS checkout replacing WAV pointer files with real audio.
- `locationd` generated rednose filter libraries were built: `libpose.so` and `libcar.so`.
- `modeld` remains blocked because compiling `driving_tinygrad.pkl` requires a real `clang` binary.
- No source code was edited.
- No simulator or OpenPilot processes are currently running.

## Recommended Next Step

Install clang on the host, then rerun the targeted tinygrad model build:

```bash
sudo apt update
sudo apt install -y clang
cd /home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-metadrive-lab/openpilot
uv run --python 3.12.12 scons -j4 openpilot/selfdrive/modeld/models/driving_tinygrad.pkl.chunkmanifest
```

After the model build succeeds, rerun:

```bash
cd /home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-metadrive-lab/openpilot
env PATH=/home/keith/Desktop/work/wingfin/AI-orchestration/openpilot-metadrive-lab/openpilot/.venv/bin:$PATH CI=1 ./openpilot/tools/sim/launch_openpilot.sh
uv run --python 3.12.12 ./openpilot/tools/sim/run_bridge.py
```

## Follow-Up Notes

- A temporary `/tmp/clang` symlink to `/usr/bin/gcc` was tested, but GCC is not compatible with tinygrad's clang invocation because it rejects `--target=x86_64-none-unknown-elf`.
- `sudo -n apt-get update` failed with `sudo: interactive authentication is required`, so clang installation needs to be run manually by the user or through an authenticated terminal.
- User installed clang manually. `driving_tinygrad.pkl.chunkmanifest` and its two chunk files are now present, so the main driving tinygrad model artifact appears to be built.
- The OpenPilot manager and MetaDrive bridge must be run in separate terminals. Running both commands in one terminal only starts the second command after the first foreground process exits or is interrupted.
- User reran manager and bridge. The bridge is intentionally running MetaDrive in `Render Mode: offscreen` because `MetaDriveBridge.should_render` is `False` and the manager is launched with `CI=1`, so no visual popup/window is expected.
- `soundd` is still crashing because the host is missing the PortAudio shared library used by Python `sounddevice`: `OSError: PortAudio library not found`. This can be avoided for simulator testing by adding `soundd` to `BLOCK`, or fixed system-wide with `sudo apt install -y libportaudio2`.
- Added `docs/metadrive-launch.md` with the two-terminal launch commands, `BLOCK=soundd` workaround, controls, and offscreen rendering note.
- Added a local `--render` CLI flag to `openpilot/tools/sim/run_bridge.py` and wired it through to `MetaDriveBridge`, allowing `run_bridge.py --render` to show the MetaDrive render window.
- Updated `docs/metadrive-launch.md` with visual and offscreen bridge commands.
- Added MetaDrive render-window key forwarding for `--render` mode. Keys `1`, `2`, `3`, `r`, `i`, `q`, and `wasd` pressed in the MetaDrive window are forwarded into the same bridge command queue used by terminal input.
- Fixed the initial render-window key forwarding crash by registering MetaDrive `engine.accept(...)` handlers after `env.reset()` initializes `env.engine`. The earlier error was `AttributeError: 'NoneType' object has no attribute 'accept'`.
- Added render-window keyboard polling with Panda3D `KeyboardButton` because mouse focus worked but event callbacks did not appear to deliver keyboard input. Held `wasd` keys repeat while held; `1`, `2`, `3`, `r`, `i`, and `q` are edge-triggered.
- Changed `--render` mode to show the ego vehicle mesh by setting `render_vehicle=True` when rendering is enabled.
- Added MetaDrive's native `KeyboardController` for `wasd` manual driving in the render window while OpenPilot is not engaged.
- `--render` then failed because comma.ai's `metadrive` `minimal` checkout did not include the default visible vehicle assets: `metadrive/assets/models/ferra/right_tire_front.gltf` and related files.
- Copied the missing `ferra` vehicle GLTF assets from the full upstream MetaDrive checkout at `openpilot-metadrive-lab/metadrive/metadrive/assets/models/ferra/` into the active comma checkout at `openpilot-metadrive-lab/metadrive-comma/metadrive/assets/models/ferra/`.
- Re-ran a short `run_bridge.py --render` smoke test. It reached `Render Mode: onscreen`, completed MetaDrive reset, and printed repeated `State: Ignition: True Engaged: False`, so the missing vehicle asset crash is resolved.
- The smoke test also printed `termios.error: (25, 'Inappropriate ioctl for device')` because it was run through `timeout` without a real interactive TTY. That is a test wrapper issue; normal user launch should be from an interactive terminal.
- User reported that render works and mouse camera controls work, but keyboard input still is not recorded.
- Checked the active process list on the real host: OpenPilot manager and `run_bridge.py --render` were running, so the keyboard issue is inside the input path rather than a missing process.
- Found that both the local render-window forwarding and MetaDrive's built-in `KeyboardController` depend on Panda3D keyboard state/events. Mouse working does not prove Panda3D is receiving keyboard events.
- Changed the simulator bridge to drain all queued control messages each tick instead of reading only one message.
- Changed manual `wasd` inputs to persist for 0.2 seconds after a key event so a single keypress is visible to the simulator instead of lasting only one 100 Hz tick.
- Added `--input-debug` to `run_bridge.py`. In render mode it prints Panda3D keyboard focus/state, detected window keys, forwarded commands, and MetaDrive native keyboard-controller actions.
- Syntax check passed for the edited simulator files with `python -m py_compile`.
- User reran `run_bridge.py --render --input-debug`; MetaDrive rendered and started, but the bridge crashed with `NameError: name 'time' is not defined` in `openpilot/tools/sim/bridge/common.py` after the manual-control persistence change.
- Fixed the crash by adding `import time` to `common.py`.
- Re-ran `python -m py_compile` for the edited simulator files; syntax check passed.
- User reran input debug and saw repeated `[sim-input] panda keyboard=unknown keys=[]`, confirming the MetaDrive render window still was not reporting keyboard state even though mouse camera input worked.
- Added a separate Tk-based `--control-window` mode in `run_bridge.py`. It opens an `openpilot sim controls` window with buttons and key bindings for `1`, `2`, `3`, `r`, `i`, `q`, and `wasd`, sending commands directly to the simulator queue without depending on Panda3D keyboard events.
- Updated `docs/metadrive-launch.md` to use `--render --control-window` as the preferred visual launch command.
- Re-ran `python -m py_compile` for the edited simulator files; syntax check passed.
