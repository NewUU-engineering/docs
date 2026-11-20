# SONIC sim2sim in MuJoCo (Unitree G1)

This guide describes how to run NVIDIA GEAR's SONIC whole-body controller with a Unitree G1 model in MuJoCo, first tracking the bundled reference motions and then using the kinematic planner. It follows the [official GR00T-WholeBodyControl Quick Start](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/quickstart.html) and [keyboard tutorial](https://nvlabs.github.io/GR00T-WholeBodyControl/tutorials/keyboard.html).

The simulation runs in two processes: a Python MuJoCo simulator and the C++ SONIC deployment executable. Use two terminal windows. The MuJoCo process runs on the host; the deployment process can run natively on the host or in the project's Docker environment.

## Start sim2sim

Open two terminals in the repository. Start the simulator first.

**Terminal 1 — MuJoCo simulator (host, repository root):**

```bash
cd GR00T-WholeBodyControl
source .venv_sim/bin/activate
python gear_sonic/scripts/run_sim_loop.py
```

Wait for the MuJoCo viewer to open with the G1 model.

**Terminal 2 — keyboard deployment (host, `gear_sonic_deploy/`):**

```bash
cd GR00T-WholeBodyControl/gear_sonic_deploy
source scripts/setup_env.sh
bash deploy.sh --input-type keyboard sim
```

If Terminal 2 is in Docker, use the repository path inside the container and run the same deployment command there. The simulator in Terminal 1 remains on the host.

## Mode 1: Track bundled reference motions

The program starts in Normal Mode, which plays the preloaded reference motions.

1. In Terminal 2, press `]` to start the policy.
2. Click the MuJoCo viewer so it has keyboard focus, then press `9` to drop the simulated robot onto the ground.
3. Return focus to Terminal 2 and press `T` to play the selected motion to completion.
4. Press `N` for the next motion or `P` for the previous motion; press `T` to play the selection.
5. Press `T` again after playback to replay. Press `R` during playback to restart the current motion at its first frame without exiting the controller.
6. Press `Q` or `E` to adjust reference heading left or right. Press `I` to reset the base orientation and heading.

## Mode 2: Kinematic planner (keyboard)

Switch from Normal Mode by pressing `ENTER` in Terminal 2. The terminal prints `Planner enabled` when the planner is active. Press `ENTER` again to return to reference-motion mode.

| Key | Action |
|---|---|
| `W` / `S` | Move forward / backward |
| `A` / `D` | Move while turning left / right |
| `,` / `.` | Strafe left / right |
| `Q` / `E` | Turn the facing direction in place |
| `N` / `P` | Next / previous motion set |
| `1`–`8` | Select a mode within the current set |
| `9` / `0` | Decrease / increase movement speed |
| `-` / `=` | Lower / raise height in non-standing modes |
| `R`, backtick, or `~` | Immediately clear planner movement momentum |
| `ENTER` | Return to reference-motion mode |
| `O` | Stop control and exit |

The default motion sets include locomotion, squat/ground, boxing, and styled walking. In the locomotion set, `1`, `2`, and `3` select slow walk, walk, and run. Keyboard movement uses momentum: after a direction press, the robot gradually decelerates. For lateral movement, the official guide recommends keeping target speed near 0.4 m/s to reduce foot collisions.

## Gamepad controls

The gamepad tutorial is for a physical Unitree wireless gamepad on the real G1. Launch it from `gear_sonic_deploy/` with:

```bash
bash deploy.sh --input-type gamepad real
```

Press `Start` to begin. In reference tracking, `A` plays a motion, `L1`/`R1` select the previous/next motion, and `B` restarts the current motion from its first frame. Press `F1` to switch to planner mode. In planner mode, use the left stick for movement direction and the right stick to turn; `L1`/`R1` cycle modes, and hold `L2`/`R2` to lower/raise speed or height. `B` pauses planner motion. Press `Select` to stop control and exit.

**Do not use `--input-type gamepad` for MuJoCo sim2sim.** Use `--input-type keyboard` for the simulation. See the [official gamepad controls](https://nvlabs.github.io/GR00T-WholeBodyControl/tutorials/gamepad.html) for the full button and mode reference.

## Stop the simulation

Press `O` in the deployment terminal to stop control and exit. Then close the MuJoCo viewer or stop Terminal 1 with `Ctrl+C`.

## References

- [GR00T-WholeBodyControl repository](https://github.com/NVlabs/GR00T-WholeBodyControl)
- [Installation (Deployment)](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/installation_deploy.html)
- [Quick Start: Sim2Sim in MuJoCo](https://nvlabs.github.io/GR00T-WholeBodyControl/getting_started/quickstart.html)
- [Keyboard controls](https://nvlabs.github.io/GR00T-WholeBodyControl/tutorials/keyboard.html)
- [Gamepad controls](https://nvlabs.github.io/GR00T-WholeBodyControl/tutorials/gamepad.html)
- [Troubleshooting](https://nvlabs.github.io/GR00T-WholeBodyControl/user_guide/troubleshooting.html)
