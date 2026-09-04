# FSDS Simulator + ROS2 Bridge — Team Guide

Quick reference for launching the simulator and ROS2 bridge on this machine.
This assumes the one-time environment setup (NVIDIA driver, ROS2 Humble,
colcon build) is already done — if you're bringing up a **new** machine from
scratch, or hit a crash that looks unfamiliar, see `SETUP_DEBUG_LOG.md` first
(full history of every bug hit and how it was fixed).

## Layout

```
A1 Challenge Simulator/
├── Formula-Student-Driverless-Simulator/   # cloned repo (source of truth)
│   ├── settings.json                       # vehicle/sensor config — edit this
│   ├── ros2/                                # built colcon workspace
│   └── maps/YonginCircuit/                  # custom track (see maps/YonginCircuit/README.md)
├── simulator/                               # prebuilt v2.2.0 binary
│   └── FSDS.sh                              # launch this
└── SETUP_DEBUG_LOG.md / SIMULATOR_GUIDE.md  # docs (this file)
```

`~/Formula-Student-Driverless-Simulator` is a **symlink** to the repo folder
above (matches what the upstream docs and the ROS2 launch file expect). Don't
recreate it as a real directory — if it's ever missing, restore it with:
```bash
ln -s "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator" \
      ~/Formula-Student-Driverless-Simulator
```

## 1. Launch the simulator

```bash
cd "/home/ailab/git/A1 Challenge Simulator/simulator"
./FSDS.sh
```
That's it — windowed mode is now the default (baked into `FSDS.sh`), and
`settings.json` is found automatically via the `$HOME` symlink above.

This opens the main menu; click through to **Play** for the default training
track, or select **CustomMap** and paste a track CSV path to load a specific
track from the GUI.

### Loading a custom track (e.g. the Yongin circuit) from the command line

```bash
./FSDS.sh -CustomMapPath=/home/ailab/track_yongin.csv
```
This skips the menu and loads the track directly.

**Important**: the path after `-CustomMapPath=` must not contain spaces (this
project's folder name does) — command-line args get mangled by Unreal's
internal re-tokenizing if they do. Keep a spaceless copy handy:
```bash
cp "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator/maps/YonginCircuit/track_yongin.csv" \
   /home/ailab/track_yongin.csv
```
(Loading via the GUI's CustomMap text field doesn't have this restriction —
that's a text box, not a command-line arg.)

## 2. Launch the ROS2 bridge

Two ways, pick based on what you need:

### A. Sensors only (IMU/GPS/GSS/lidar, no cameras)
```bash
source /opt/ros/humble/setup.bash
source "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator/ros2/install/setup.bash"
ros2 run fsds_ros2_bridge fsds_ros2_bridge
```
Topics come up **unnamespaced**: `/gps`, `/imu`, `/gss`, `/lidar/Lidar1`,
`/lidar/Lidar2`, `/control_command`, `/testing_only/odom`,
`/testing_only/track`, `/signal/go`, `/signal/finished`, `/tf_static`,
`/wheel_states`.

### B. Full stack including cameras (recommended)
```bash
source /opt/ros/humble/setup.bash
source "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator/ros2/install/setup.bash"
ros2 launch fsds_ros2_bridge fsds_ros2_bridge.launch.py
```
Spawns one extra node per camera defined in `settings.json` (currently `cam1`,
`cam2`), and everything comes up properly namespaced under `/fsds/...`:
`/fsds/gps`, `/fsds/imu`, `/fsds/cam1/image_color`, `/fsds/cam1/camera_info`,
`/fsds/lidar/Lidar1`, etc. This relies on the `$HOME` symlink above to find
`settings.json`.

Either way, you should see in the terminal:
```
[INFO] [fsds_ros2_bridge]: Connected to the simulator!
...
GPS enabled
GSS enabled
IMU enabled
```
That confirms the bridge found and connected to a running simulator instance
(launch the simulator **first**, bridge second).

## 3. Sanity-check it's actually working

```bash
ros2 topic list
ros2 topic hz /imu          # or /fsds/imu if launched via method B — should read ~250Hz
ros2 topic echo /gps --once # or /fsds/gps
```
For a custom map, cross-check the simulator's actual cone list against what
you expect (don't trust visuals alone — see `maps/YonginCircuit/README.md` for
why):
```bash
ros2 topic echo /testing_only/track --once
```

## 4. Changing the vehicle/sensor config

Edit **`Formula-Student-Driverless-Simulator/settings.json`** directly — it's
the single source of truth (both the simulator and the `ros2 launch` camera
spawner read it via the `$HOME` symlink; the bridge's own sensor topics are
pulled live from the simulator over RPC, not from a file, so they always match
whatever the simulator loaded). Restart both the simulator and the bridge for
changes to take effect. Supported sensor types: `Imu`(2), `Gps`(3),
`Distance`(5), `Lidar`(6), `GSS`(7) under `Sensors`; cameras go under the
separate `Cameras` block. Adding a sensor there is enough — no code/rebuild
needed, verified empirically by adding a test camera + lidar and confirming
new topics with live data.

## Known gotchas (see `SETUP_DEBUG_LOG.md` for full details)

- **Fresh NVIDIA driver installs**: only `nvidia-driver-470-server` has been
  verified stable with this UE4.27-based simulator — newer branches (595+)
  crash with `VK_ERROR_DEVICE_LOST`.
- **Custom map cone cap**: `CustomMap` silently caps around **128 total
  cones** (confirmed via `/testing_only/track`, not just visually) and its
  render/world extent is sized for compact FS-style tracks, not real-world-
  scale circuits. See `maps/YonginCircuit/README.md` if building a new custom
  track.
- **`-settings=`/`-CustomMapPath=` and spaces don't mix** — always use
  spaceless paths for command-line args (the `$HOME` symlink already handles
  `settings.json` discovery for you; this only matters for `-CustomMapPath`).
