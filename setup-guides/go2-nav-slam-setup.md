# Unitree Go2 EDU: Navigation & SLAM Stack (THD GO2_THDTCC) — Current Setup

> **Status:** work in progress (November 2026). This guide records what has been set up and changed so far so others can connect and continue. Things will move; update this file when they do.
>
> For the RL policy deployment environment on the same robot (unitree_sdk2, ONNX Runtime), see [OnBoardJetsonSetup.md](OnBoardJetsonSetup.md). **Do not run the RL low-level controller and this navigation stack at the same time**: low-level control takes the legs away from Unitree's sport mode, which this stack drives through `cmd_vel`.

## 1. What this system is

We are porting the THD (Deggendorf) Go2 inspection/navigation stack. It has two halves:

| Part | Repo | Runs on | OS / ROS |
|---|---|---|---|
| Host (SLAM, Nav2, exploration, RViz) | `thd-research/GO2_THDTCC` (private; README still says `adhamaziz/...`) | Laptop, inside Docker | Ubuntu 22.04 + ROS 2 Humble (container `go2_humble_container`) |
| Onboard (robot driver, lidar, TF) | `thd-research/GO2_ONBOARD_THDTCC` | Go2 Jetson | Ubuntu 20.04 + ROS 2 **Foxy** |

The two sides talk through a **Zenoh bridge** (`zenoh-bridge-ros2dds`), which also bridges Foxy ↔ Humble:

```
Jetson (Foxy, ROS_DOMAIN_ID 0)                    Laptop container (Humble, ROS_DOMAIN_ID 1)
go2_bringup: go2_driver + Delta-2A + TF            slam_toolbox / Nav2 / m-explore / RViz
        │                                                       │
 robot zenoh bridge  ── Ethernet, 192.168.123.18:7447 ──  PC zenoh bridge
```

The robot bridge only forwards the topics listed in `zenoh_bridge/robot_config.json5` (`/scan`, `/tf`, `/tf_static`, `/odom`, `/robot_description`, `/joint_states`, `/cmd_vel`, …).

**Lidar:** the THD authors used a Hesai XT32 on top of the robot (the Go2's built-in L1 was unreliable for them). **We use a 3iRobotics Delta-2A 2D lidar instead**, with our own driver: [krranky/delta2a_lidar_ros2](https://github.com/krranky/delta2a_lidar_ros2). It publishes `/scan` directly.

There is **no RL or learned locomotion** in this stack. Walking is Unitree's built-in sport-mode controller; the only learned model is the (not yet set up) onboard YOLO detector.

---

## 2. Changes from the original code

Everything we had to change to get it running. If you clone fresh, re-apply these.

### 2.1 Both sides
| Change | Why |
|---|---|
| In `src/go2_robot/`, rename the root `package.xml` → `package.xml.bak` and `CMakeLists.txt` → `CMakeLists.txt.bak` | The submodule root declares a duplicate `go2_description` package. colcon stops descending there, so it only finds 1 package instead of 10 (`go2_driver`, `go2_bringup`, `go2_nav`, …). |
| Submodules cloned by hand (see §4, §5) | GitHub zip downloads leave submodule folders empty. The submodule repos under `adhamaziz/` are public. |
| `chmod +x` on the Zenoh launch scripts | Executable bit lost when extracting the zip. |

### 2.2 Onboard (Jetson)
| Change | Why |
|---|---|
| `touch src/hesai_lidar/COLCON_IGNORE` | Hesai not used. (If ever needed: the pinned SDK doesn't compile against the driver. Check out SDK tag `v2.0.11` in `src/hesai_lidar/src/driver/HesaiLidar_SDK_2.0`.) |
| Added package `src/delta2a_lidar` from our driver repo | Replaces the Hesai lidar. |
| Patched `delta2a_lidar/lidar_node.py` (fixed-size scans) | Original published a variable number of readings per scan with non-uniform angles; slam_toolbox rejects that. Patched version: 720 fixed bins over [-π, π) with `inf` for no return, `invert` parameter, measured `scan_time`. **Check the repo has this version:** `grep -c num_bins .../lidar_node.py` must be > 0. |
| `go2_robot/go2_bringup/launch/go2.launch.py` | Hesai include and `pointcloud_to_laserscan` removed; added a `delta2a_lidar` node and a `static_transform_publisher` `base_link → laser`. Current values: `invert: True` (our Delta spins clockwise, left/right were mirrored), `range_min: 0.15`, `range_max: 8.0`. **The base_link → laser offsets are placeholders (zeros) until measured.** |
| `~/.bashrc`: appended `[ -f ~/go2_ws/install/setup.bash ] && source ~/go2_ws/install/setup.bash` | Foxy + CycloneDDS were already sourced by a fixed block (no fishros prompt on our unit). |
| User `unitree` added to `dialout` and `docker` groups | Serial port access for the lidar; running the Zenoh Docker container without sudo. |
| Installed apt packages | `nlohmann-json3-dev ros-foxy-realsense2-camera python3-serial ros-foxy-xacro ros-foxy-robot-state-publisher` (`ros-foxy-pointcloud-to-laserscan` was already there). |

### 2.3 Host (laptop)
| Change | Why |
|---|---|
| `mapper_params_online_async.yaml`: `max_laser_range: 8.0` (was 15.0) | Delta-2A range is 8 m. |
| Docker: installed `docker-compose-v2` + `docker-buildx` (or `docker-compose-plugin` + `docker-buildx-plugin` with docker-ce) | Ubuntu's `docker.io` ships without them; `docker compose` fails otherwise. |
| In the container (not persisted unless added to the Dockerfile) | `ros-humble-teleop-twist-keyboard`, optionally `ros-humble-joint-state-publisher-gui`, `mesa-utils` |

---

## 3. Network setup (laptop ↔ Go2)

| Device | IP |
|---|---|
| Laptop (Ethernet) | `192.168.123.100/24` (static) |
| Jetson | `192.168.123.18`, user `unitree` (ask the team for the password; don't commit it) |

The Jetson has no internet of its own. The laptop shares its Wi-Fi over the Ethernet cable. **The route/DNS on the Jetson and the NAT rules on the laptop reset on every reboot**, so we use two scripts.

### 3.1 One-time: static IP profile on the laptop
```bash
nmcli device        # find your wired adapter (USB adapters are named enx...)
sudo nmcli con add type ethernet con-name go2-lan ip4 192.168.123.100/24
# if another wired profile keeps trying DHCP on that adapter:
sudo nmcli con mod "Wired connection 1" connection.autoconnect no
```

### 3.2 One-time: laptop script `~/go2_net.sh`
Set `WIFI` to your Wi-Fi interface (`nmcli device`). The Ethernet adapter is detected automatically.
```bash
cat > ~/go2_net.sh << 'EOF'
#!/bin/bash
set -e
WIFI=wlp87s0
ETH=$(nmcli -t -f DEVICE,TYPE device | awk -F: '$2=="ethernet"{print $1; exit}')
if [ -z "$ETH" ]; then echo "No Ethernet adapter found - plug it in"; exit 1; fi
echo "Using Ethernet: $ETH"
sudo nmcli con up go2-lan ifname $ETH
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -C POSTROUTING -s 192.168.123.0/24 -o $WIFI -j MASQUERADE 2>/dev/null || \
  sudo iptables -t nat -A POSTROUTING -s 192.168.123.0/24 -o $WIFI -j MASQUERADE
sudo iptables -C FORWARD -i $ETH -o $WIFI -j ACCEPT 2>/dev/null || \
  sudo iptables -I FORWARD -i $ETH -o $WIFI -j ACCEPT
sudo iptables -C FORWARD -i $WIFI -o $ETH -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null || \
  sudo iptables -I FORWARD -i $WIFI -o $ETH -m state --state RELATED,ESTABLISHED -j ACCEPT
ping -c 1 -W 2 192.168.123.18 > /dev/null && echo "Robot reachable. Now: ssh unitree@192.168.123.18 and run ./net_up.sh" \
  || echo "Laptop configured, but robot not reachable - is it powered on and cabled?"
EOF
chmod +x ~/go2_net.sh
```
The FORWARD rules are required because Docker sets the default forward policy to DROP.

### 3.3 One-time: Jetson script `~/net_up.sh`
```bash
cat > ~/net_up.sh << 'EOF'
#!/bin/bash
sudo ip route replace default via 192.168.123.100 dev eth0
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf > /dev/null
ping -c 2 -W 2 8.8.8.8 > /dev/null && echo "Internet: OK" || echo "Internet: FAILED (run ~/go2_net.sh on the laptop)"
ping -c 1 -W 3 github.com > /dev/null && echo "DNS: OK" || echo "DNS: FAILED"
EOF
chmod +x ~/net_up.sh
```

> Run laptop commands on the laptop (`azim@...`) and Jetson commands over SSH (`unitree@ubuntu`). Mixing them up was our most common mistake.

---

## 4. Host side (laptop) — first-time setup

```bash
# Repo is private: ask for access to thd-research/GO2_THDTCC, then
gh auth setup-git
git clone --recurse-submodules https://github.com/thd-research/GO2_THDTCC.git
# Without access: unzip the shared zip into ~/GO2_THDTCC and fill submodules:
#   cd ~/GO2_THDTCC/src && rmdir go2_robot m-explore-ros2
#   git clone -b humble https://github.com/adhamaziz/go2_robot_THDTCC.git go2_robot
#   git clone https://github.com/adhamaziz/m-explore-ros2.git m-explore-ros2

cd ~/GO2_THDTCC/src/go2_robot && mv package.xml package.xml.bak && mv CMakeLists.txt CMakeLists.txt.bak
chmod +x ~/GO2_THDTCC/src/zenoh_bridge_ros2dds/launch_zenoh.sh

cd ~/GO2_THDTCC
xhost +local:root
docker compose build
docker compose up -d
docker exec -it go2_humble_container bash
# inside:
cd /ros2_ws && colcon list           # should list go2_bringup, go2_driver, go2_nav, explore_lite, ...
colcon build --symlink-install
source install/setup.bash
```

Quick test without the robot: view the URDF with `ros2 launch go2_description robot.launch.py use_sim_time:=False`, `ros2 run joint_state_publisher_gui joint_state_publisher_gui`, `ros2 launch go2_rviz rviz.launch.py`, and set RViz Fixed Frame to `base_link`. **Stop these before connecting to the real robot**, or the fake joint states fight the real TF.

---

## 5. Onboard side (Jetson) — first-time setup

Workspace must be at **`/home/unitree/go2_ws`** (paths are hardcoded in the launch files).

```bash
ssh unitree@192.168.123.18
~/net_up.sh

# get the repo (or scp the zip from the laptop and unzip + mv GO2_ONBOARD_THDTCC-main go2_ws)
cd ~/go2_ws/src
rmdir go2_robot hesai_lidar go2_aruco go2_yolo
git clone -b humble https://github.com/adhamaziz/go2_robot_THDTCC.git go2_robot
git clone --recurse-submodules https://github.com/adhamaziz/HesaiLidar_ROS_2.0.git hesai_lidar
git clone https://github.com/adhamaziz/GO2_ARUCO_THDTCC.git go2_aruco
git clone https://github.com/adhamaziz/GO2_YOLO_THDTCC.git go2_yolo
git clone https://github.com/krranky/delta2a_lidar_ros2.git delta2a_tmp
mv delta2a_tmp/src/delta2a_lidar . && rm -rf delta2a_tmp

cd go2_robot && mv package.xml package.xml.bak && mv CMakeLists.txt CMakeLists.txt.bak && cd ..
touch hesai_lidar/COLCON_IGNORE
# replace go2_robot/go2_bringup/launch/go2.launch.py with our Delta-2A version (see §2.2)

sudo apt install -y nlohmann-json3-dev ros-foxy-realsense2-camera python3-serial \
                    ros-foxy-xacro ros-foxy-robot-state-publisher
sudo usermod -aG dialout,docker unitree      # then log out and back in
echo '[ -f ~/go2_ws/install/setup.bash ] && source ~/go2_ws/install/setup.bash' >> ~/.bashrc

cd ~/go2_ws
MAKEFLAGS=-j2 colcon build --symlink-install --parallel-workers 2 \
  --packages-up-to go2_bringup delta2a_lidar
source install/setup.bash
chmod +x ~/go2_ws/zenoh_bridge/launch.sh
```

Limit to 2 build jobs on the Jetson to avoid running out of RAM. `go2_aruco` and `go2_yolo` are Docker-only and are skipped by colcon automatically.

### Lidar hardware
- Delta-2A connects through its CP2102 USB-serial adapter to a USB port on the expansion dock (shows up as `/dev/ttyUSB0`; prefer the stable `/dev/serial/by-id/...` path in `go2.launch.py`).
- Powered the same way as when tested on a laptop. Never feed the lidar from the dock's power output directly (battery voltage).
- Mount on top of the body behind the head with a clear 360° view; route the cable away from the legs.

---

## 6. Every session

**Laptop**
```bash
~/go2_net.sh
ssh unitree@192.168.123.18
```

**Jetson** (three SSH terminals)
```bash
# T1
~/net_up.sh
ros2 launch go2_bringup go2.launch.py realsense:=False
# T2: robot Zenoh bridge
cd ~/go2_ws/zenoh_bridge && ./launch.sh
# T3: sanity check
ros2 topic hz /scan            # ~6-7 Hz
ros2 topic echo /odom --once
```

**Laptop**
```bash
# T4: PC Zenoh bridge (on the host, not in the container)
cd ~/GO2_THDTCC/src/zenoh_bridge_ros2dds && ./launch_zenoh.sh
# T5: container
xhost +local:root
cd ~/GO2_THDTCC && docker compose up -d
docker exec -it go2_humble_container bash
source install/setup.bash
ros2 topic list                # /scan, /odom, /tf, /joint_states should be here
rviz2
```

### RViz setup
| Display | Settings |
|---|---|
| Global Options | Fixed Frame `odom` (or `base_link` to pin the body; `laser` to view the raw scan with no TF lookup) |
| LaserScan | Topic `/scan` (Reliability `Best Effort` if empty) |
| RobotModel | Source `Topic`, `/robot_description`, **Durability `Transient Local`**. Fallback: Source `File`, `/ros2_ws/install/go2_description/share/go2_description/urdf/go2_description.urdf` |
| Axes | Reference Frame `base_link`. Red = +x (forward), green = +y (left), blue = up. Grid cells are 1 m. |
| TF | Optional; untick most leg frames to reduce clutter |

The model follows the real legs (`/joint_states` from `go2_driver`). Test by standing up / lying down with the remote.

---

## 7. Lidar calibration (do before mapping)

In RViz with Fixed Frame `base_link` and the Axes display:

1. **Direction:** box on the robot's left must appear on the green-arrow side. If mirrored, toggle `invert` in `go2.launch.py`. (Ours: `True`, done.)
2. **Front:** box in front must appear along the red arrow. If rotated, set the yaw in `laser_tf_cmd`.
3. **Position:** measure x, y, z of the lidar from the body centre (m) and put them in `laser_tf_cmd` (Foxy positional order: `x y z yaw pitch roll base_link laser`). **Not done yet.**
4. **Self-hits:** raise `range_min` if the robot's body appears in the scan.

`--symlink-install` means launch file edits apply after restarting the bringup, no rebuild.

---

## 8. SLAM (mapping) — next step, not yet tested on our setup

Robot bringup and both bridges running. In the container:

```bash
ros2 launch slam_toolbox online_async_launch.py use_sim_time:=false \
  slam_params_file:=/ros2_ws/mapper_params_online_async.yaml
```
RViz: Fixed Frame `map`, add Map display on `/map`.

Drive manually (`/cmd_vel` is bridged to the robot):
```bash
apt install -y ros-humble-teleop-twist-keyboard
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```
- Robot standing in normal walking mode (not stand lock), remote in hand as an emergency stop.
- Press `z` repeatedly to ~0.2 m/s and ~0.4 rad/s before driving. `k` stops.
- Walk slowly, turn slower (Delta-2A gives ~6.5 scans/s over 8 m), stay within a few metres of walls, finish where you started for loop closure.

Save:
```bash
mkdir -p /ros2_ws/maps
ros2 run nav2_map_server map_saver_cli -f /ros2_ws/maps/lab_map
```

After that, follow the THD two-phase workflow in `GO2_THDTCC/OPERATIONS_README.md` §8: map once, then localize (AMCL) + navigate on the saved map. Frontier exploration (`nav_slam_launch.py` + `explore_launch.py`) comes after manual mapping works.

---

## 9. Troubleshooting & known issues

| Symptom | Cause / fix |
|---|---|
| `ping 8.8.8.8` on Jetson: *Destination Host Unreachable* | Jetson default route reset to `192.168.123.1` after reboot → `~/net_up.sh`. If still failing, laptop NAT is gone → `~/go2_net.sh`. |
| `Temporary failure in name resolution` | `/etc/resolv.conf` reset → `~/net_up.sh`. |
| `nmcli con up go2-lan`: *No suitable device found* | Ethernet adapter unplugged, or different adapter name. Script auto-detects; check `nmcli device`. |
| `git clone ... Repository not found` | Host repo is private (and moved from `adhamaziz` to `thd-research`). Get access, then `gh auth setup-git`. |
| `docker build requires 1 argument` / `docker compose` unknown | Compose/buildx plugins missing (§2.3). |
| `colcon list` shows only `go2_description` | Root `package.xml` in `go2_robot` not moved aside (§2.1). |
| `Package 'go2_bringup' not found` | Workspace not built, or not sourced. |
| `not found: .../hesai_ros_driver/.../local_setup.bash` | Leftover from a failed Hesai build: `rm -rf build/hesai_ros_driver install/hesai_ros_driver`, rebuild any package. |
| Launch fails: `executable ... not found on the PATH` | `xacro` missing → `sudo apt install ros-foxy-xacro`. |
| Lidar: `Permission denied: /dev/ttyUSB0` | Not in `dialout`, or session predates the group change → re-login. |
| Lidar node opens port but no `/scan` | Check the lidar spins; dump raw bytes: `stty -F /dev/ttyUSB0 230400 raw cs8 -cstopb -parenb; timeout 3 cat /dev/ttyUSB0 \| xxd \| head` (expect repeating `aa00 a6`). |
| `./launch.sh: Permission denied` | `chmod +x launch.sh`; if it mentions `docker.sock`, add user to `docker` group. |
| RViz: *Frame [odom] does not exist* | Robot driver / bridge not running, or use `base_link`. |
| RViz robot model distorted / legs missing | No `/joint_states` (real robot not connected, or no joint_state_publisher in offline test). |
| Scan in RViz lags occasionally | **Unresolved.** Clocks verified in sync (< 0.01 s). When it happens, check `ros2 topic hz /scan` on both sides and `ros2 topic delay /scan` in the container to locate it (Jetson CPU vs Zenoh vs RViz). Also check `glxinfo \| grep "OpenGL renderer"` in the container: `llvmpipe` = RViz rendering on CPU. |
| Robot collapsed while in stand lock | Motors overheat holding the robot (thermal protection) or low battery. Keep it lying down while doing software work. Check `ros2 topic echo /lowstate --once` for battery `soc` and motor temperatures. |
| Battery dead → Jetson off | The Jetson is powered from the robot battery; no supported external power. Charge and continue. |

---

## 10. Open items

- [ ] Measure and set the `base_link → laser` transform.
- [ ] Push our modified `go2.launch.py` and patched `lidar_node.py` to a team fork (currently only on the Jetson / in this guide).
- [ ] First SLAM map with teleop; then AMCL + Nav2; then frontier exploration.
- [ ] Find the cause of the occasional scan lag.
- [ ] Make the Jetson's gateway/DNS permanent (NetworkManager profile → gateway `192.168.123.100`) instead of `net_up.sh`.
- [ ] Not started: RealSense, YOLO, ArUco containers (perception / SWAP inspection pipeline).
- [ ] Optional later: 3D LIO (Point-LIO / FAST-LIO2) with the built-in L1, projected to 2D for Nav2.
