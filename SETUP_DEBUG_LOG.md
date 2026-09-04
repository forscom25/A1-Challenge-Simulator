# FSDS Simulator + ROS2 Humble Bridge — Setup & Debugging Log

Date: 2026-09-02
Machine: Ubuntu 22.04.5, RTX A5000, fresh install (no git/gcc/ROS2/NVIDIA driver present at start)

## Goal

Run the FS-Driverless Formula Student Driverless Simulator together with its
`fsds_ros2_bridge` ament_cmake package against a real ROS2 Humble install, in
`/home/ailab/git/A1 Challenge Simulator/`, for later use with a real ROS2 Humble
autonomy stack.

## Layout produced

- `Formula-Student-Driverless-Simulator/` — clone of
  `FS-Driverless/Formula-Student-Driverless-Simulator` (`master`), submodules
  `AirSim/external/rpclib` and `ros2/src/fs_msgs` (ros2 branch) initialized.
  `ros2/` here holds the built colcon workspace (`fs_msgs`, `fsds_ros2_bridge`).
- `simulator/` — unzipped prebuilt Linux binary release `v2.2.0`
  (`fsds-v2.2.0-linux.zip`), used instead of building Unreal Engine from source.

## Setup steps

1. Base tooling: `git`, `git-lfs`, `build-essential`, `cmake`, `libcurl4-openssl-dev`,
   `python3-pip`, etc.
2. NVIDIA driver via `ubuntu-drivers autoinstall` (see Bug 1 below — this had to be
   redone).
3. ROS2 Humble desktop via the official apt repo, plus `ros-dev-tools`,
   `python3-colcon-common-extensions`, `python3-rosdep`.
4. Cloned the repo with `--recurse-submodules`.
5. Downloaded/unzipped the `v2.2.0` Linux simulator binary.
6. `AirSim/setup.sh` prerequisites installed manually (Ubuntu 22.04 branch of the
   script wants `clang-12`/`libc++-12-dev`/`libc++abi-12-dev`; downloaded Eigen
   3.3.7 into `AirLib/deps/eigen3`).
7. `rosdep install --from-paths src --ignore-src -r -y` in `ros2/` — everything was
   already satisfied by `ros-humble-desktop` + `libcurl4-openssl-dev`.
8. `colcon build` in `ros2/` — succeeded using the system default `g++` (11.4);
   clang-12 was only needed to satisfy `setup.sh`, not the actual bridge build.

All of the above went smoothly. Two real bugs came up when actually running the
simulator.

## Bug 1: NVIDIA driver 595 too new for this UE4.27-based simulator

**Symptom:** simulator process crashed a few seconds after reaching the main menu
level, with:
```
LogVulkanRHI: Error: VulkanRHI::vkDeviceWaitIdle(Device) failed, VkResult=-4
... VK_ERROR_DEVICE_LOST
Signal=11 (segfault)
```

**Investigation:**
- `ubuntu-drivers autoinstall` had installed driver `595.84` (very new/bleeding-edge
  at the time).
- The simulator is built on Unreal Engine 4.27 (~2021), whose Vulkan RHI predates
  that driver by years.
- Tried `-opengl4` as a cheap workaround: got further (loaded the actual track level
  instead of crashing at the menu), but then aborted with `SIGABRT` inside
  `__cxa_throw` — the packaged binary's shaders were only cooked for Vulkan, so
  forcing OpenGL isn't a real option for a non-editor packaged build.
- Web/GitHub search confirmed `VK_ERROR_DEVICE_LOST` on Linux is a known class of
  issue for Unreal Engine + very new NVIDIA drivers (reported across UE4 AirSim,
  UE5.7/5.8, CARLA, etc.), generally resolved by using an older driver branch.

**Fix:** purged the 595 driver and installed `nvidia-driver-470-server` instead
(a long-term branch from roughly UE4.27's own era, still fully supports the
Ampere-based RTX A5000), then rebooted.
```bash
sudo apt purge -y '^nvidia-.*'
sudo apt autoremove -y
sudo apt install -y nvidia-driver-470-server
sudo reboot
```
After this the simulator ran stably under the default Vulkan RHI.

## Bug 2: settings.json fails to parse because the project path contains spaces

**Symptom:** after Bug 1 was fixed, the simulator reached the main menu fine, but
crashed immediately when transitioning into the actual track level:
```
libc++abi: terminating with uncaught exception of type std::invalid_argument:
Error while parsing settings.json: parse error - unexpected '"'
```
even though the `settings.json` file itself is valid JSON (verified with
`python3 -c "import json; json.load(...)"`).

**Root cause:** the project directory is named `A1 Challenge Simulator` (contains
spaces). The simulator was launched with an explicit
`-settings "/home/ailab/git/A1 Challenge Simulator/.../settings.json"` argument.
Unreal Engine's Linux command-line handling rejoins argv into a single string and
re-tokenizes it internally; a path containing unescaped spaces gets split into
multiple tokens at that stage, so the `-settings` flag ends up pointing at a
mangled/partial value instead of the real file — which then fails to parse as
either a path or an inline JSON string.

**Fix (original, 2026-09-02):** stopped passing `-settings` entirely. Instead,
copied `settings.json` into the same directory as `FSDS.sh`
(`simulator/settings.json`) and launched the binary from that directory, relying
on the simulator's current-working-directory lookup (one of its documented
settings-discovery locations) instead of a command-line argument. This sidesteps
the argv space-splitting bug entirely — no `$HOME` symlink or relocation was used.

**Superseded 2026-09-03:** switched to the `$HOME`-symlink approach the docs
actually recommend (see "Update 2026-09-03" below) — the CWD-copy fix above
still works and the reasoning stays valid, but is no longer what's in place.

## Bug 3: fullscreen (default) launch crashes with VK_ERROR_INITIALIZATION_FAILED

**Symptom:** `./FSDS.sh` with no arguments (defaults to fullscreen) crashes a
few seconds in:
```
LogCore: Fatal error: [File:.../VulkanSwapChain.cpp] [Line: 538]
Result failed, VkResult=-3 ... with error VK_ERROR_INITIALIZATION_FAILED
Signal=11 (segfault)
```
Reproduced consistently (2026-09-03, after a reboot) — not transient. Driver is
still the working 470.256.02 branch (Bug 1 fix intact), display/X session is
fine (`xrandr`/`nvidia-smi` both healthy, GPU actively rendering the desktop).

**Fix (original, 2026-09-03):** always pass `-windowed` (with `-ResX`/`-ResY`):
```bash
./FSDS.sh -windowed -ResX=1280 -ResY=720
```
This has been reliable both today and in the original setup session — only the
argument-less (fullscreen) path crashes. Root cause not fully diagnosed (likely
a fullscreen-exclusive/compositor interaction on this single-monitor Vulkan
setup); windowed mode sidesteps it entirely, so no further investigation done.

**Superseded 2026-09-03 (same day):** `simulator/FSDS.sh` itself was patched so
plain `./FSDS.sh` (no args) now always launches windowed — see "Update
2026-09-03" below. The flags above still work identically if passed explicitly
(they're now redundant with the script's own default, not conflicting).

## Update 2026-09-03: `$HOME` symlink + `FSDS.sh` defaults to windowed

Two workflow changes, both requested to match documented FSDS behavior more
closely and simplify day-to-day launching:

**1. `$HOME` symlink** (supersedes Bug 2's original CWD-copy fix): FSDS's docs
assume the repo is cloned into `~/Formula-Student-Driverless-Simulator` and
look for `settings.json` there first. Since this project instead lives at
`/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator`
(a path with spaces, which is *why* Bug 2 happened), created a symlink instead
of a second copy:
```bash
ln -s "/home/ailab/git/A1 Challenge Simulator/Formula-Student-Driverless-Simulator" \
      ~/Formula-Student-Driverless-Simulator
```
Verified this works standalone (removed `simulator/settings.json` entirely,
confirmed via the ROS2 bridge that sensor config still loads correctly through
the symlink alone) — so `simulator/settings.json` is no longer needed or kept;
`Formula-Student-Driverless-Simulator/settings.json` (the git-tracked file) is
now the single source of truth, resolved automatically via `$HOME`. This also
means the `ros2/src/fsds_ros2_bridge/launch/fsds_ros2_bridge.launch.py` launch
file (which hardcodes reading `~/Formula-Student-Driverless-Simulator/settings.json`
to spawn one camera node per configured camera) now works without any manual
file copying too.

Note: this symlink only fixes *settings.json* discovery. `-CustomMapPath` still
needs a spaceless path when passed on the command line (Bug 2's argv-splitting
issue is unrelated to settings-file lookup) — keep copying custom map CSVs to
somewhere like `/home/ailab/track_*.csv` as before.

**2. `FSDS.sh` defaults to windowed**: edited `simulator/FSDS.sh` (the launcher
script from the downloaded release zip, not git-tracked) to always inject
`-windowed -ResX=1280 -ResY=720` ahead of `"$@"`:
```sh
"$UE4_PROJECT_ROOT/FSOnline/Binaries/Linux/Blocks" FSOnline -windowed -ResX=1280 -ResY=720 "$@"
```
So `./FSDS.sh` alone now works (Bug 3 is a non-issue without remembering the
flag), while `./FSDS.sh "-CustomMapPath=..."` etc. still passes extra args
through unchanged. **Caveat**: since this edits the file that ships inside the
downloaded release zip, re-unzipping a fresh `fsds-vX.Y.Z-linux.zip` in the
future will overwrite this fix — reapply the one-line edit above if that
happens.

## Verified working state

- Simulator: `simulator/FSDS.sh "-CustomMapPath=/home/ailab/track_yongin.csv"`
  (windowed is now automatic), main menu clicked through manually to load the
  track (or `-CustomMapPath` skips the menu entirely).
- Bridge: `ros2 run fsds_ros2_bridge fsds_ros2_bridge` connects immediately and
  publishes `/gps`, `/imu` (~249Hz), `/gss`, `/lidar/Lidar1`, `/lidar/Lidar2`,
  `/testing_only/odom` (~249Hz), `/testing_only/track`, `/signal/go`, etc.
  (`ros2 launch fsds_ros2_bridge fsds_ros2_bridge.launch.py` additionally
  starts one camera node per configured camera, under `/fsds/...`.)
- Note: `ros2 run` gives unnamespaced topics (e.g. `/gps`); `ros2 launch` gives
  the `/fsds/...` namespacing the (ROS1-written) docs describe.

## Note: `fsds_ros2_bridge` doesn't read settings.json from disk itself

Checked `fsds_ros2_bridge`'s source
(`ros2/src/fsds_ros2_bridge/src/airsim_ros_wrapper.cpp:17`) to confirm the bridge
does **not** read its own local settings.json to decide which sensors/topics to
create. It fetches the settings text live from the running simulator over the RPC
connection (`airsim_client_.getSettingsString()`); a `readTextFromFile()` helper
with a local-file error message exists in that file but is dead code, never
called. So the bridge always mirrors whatever the connected simulator instance
actually has loaded — there's no second settings.json to keep in sync for the
bridge's own purposes (the camera *launch file* is the exception — see the
`$HOME` symlink note above, it reads settings.json itself to know which camera
nodes to spawn).
