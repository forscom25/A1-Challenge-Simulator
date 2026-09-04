# Real-Scale Yongin Circuit via Unreal Engine Source Build — Progress

Status as of 2026-09-04. Goal: build FSDS from Unreal Engine source so the
real, full-scale Yongin circuit (not the scaled-down `CustomMap` workaround)
can be used as an actual level, bypassing the ~128-cone cap and small
world-extent limits of the packaged binary's `CustomMap`.

## Done

1. **Unreal Engine 4.27 built from source** at `~/UnrealEngine` (Editor,
   ShaderCompileWorker, UnrealLightmass). Required building targets
   one-at-a-time (`Build.sh <Target> Linux Development`) — building all 7
   default targets in parallel via bare `make` hit a race in Epic's Linux
   progress-wrapper script and silently dropped 4 of 7 targets with no error.
2. **FSDS's own project compiled** (`BlocksEditor` = `Blocks` game module +
   `AirSim` plugin). Hit and fixed two real bugs along the way:
   - `AirSim/build.sh` had never been run in this project before (only the
     ROS2 bridge's separate build path had), so `AirLib` was never staged
     into `UE4Project/Plugins/AirSim/Source/AirLib` — fixed by running it.
   - **ABI mismatch**: `librpc.a` (built with the system's `clang++-12`)
     referenced `pthread_cond_clockwait`, a symbol not present in UE4's
     bundled Linux toolchain's sysroot (`Engine/Extras/.../v19_clang-11.0.1-centos7`,
     an old CentOS7-based glibc). Fixed by rebuilding `AirLib`/`rpclib` using
     UE4's own bundled `clang`/`clang++` **and** passing the matching
     `--sysroot=` — compiling against the old sysroot's headers means
     libc++'s feature detection correctly avoids that codepath, matching
     what UE4Editor's own linker links against.
3. **`UE4Editor` launches successfully** with `FSOnline.uproject` loaded,
   `TrainingMap` open. Enabled the `EditorScriptingUtilities` plugin (added
   to `FSOnline.uproject`) so Python editor-scripting (`unreal.EditorLevelLibrary`
   etc.) works — required a restart to take effect.
4. **Investigated `TrainingMap`'s ground** via Python: it's a plain
   `StaticMeshActor` named `floor` (not a Landscape), base tile 1m×1m,
   currently scaled to 200m×250m — trivially resizable via `Scale`.
5. **Investigated the `spline_cones_best_200` actor**: this is the
   documented, standard map-authoring cone-spline system (see
   `docs/map-tutorial.md`), *not* the opaque, capped Blueprint inside the
   packaged binary's `CustomMap` level. Confirmed via component count
   (192 `StaticMeshComponent`s for 30 spline points) that it auto-generates
   cones by resampling the spline at a fixed internal spacing — no hard cap
   like `CustomMap`'s ~128.
6. **Populated it with the real, full-scale Yongin centerline** (256 points
   at 10m spacing, 2563m loop, recentered so the start point is world
   origin). Python's `SplineComponent.set_spline_points()` doesn't trigger
   the Blueprint's Construction Script (that's only wired to real
   UI-driven transform edits, not exposed to Python in this engine version)
   — worked around by having the actor selected and nudged via the viewport
   move gizmo (**W**, drag). Result: **1278 cone mesh components**,
   visually confirmed in the Editor viewport — correct real track shape
   (hairpins, esses, sweeping corners all present and recognizable).
7. Cleaned up an accidental duplicate actor from an earlier UI mis-click.

## Blocked on / in progress right now

Saving this work as a **new** map (`YonginCircuitMap`), not overwriting
`TrainingMap`. No direct "Save As" in this engine's Python API, so the plan
is:
1. Run `unreal.EditorLevelLibrary.save_current_level()` in the Editor's
   Python console — this saves the current (edited) state into
   `TrainingMap.umap` **temporarily**.
2. (My side, shell-level) copy that saved file to a new name
   (`YonginCircuitMap.umap`), then `git checkout` to restore the pristine
   original `TrainingMap.umap` (it's git-tracked and currently clean, so
   this is fully safe/reversible).
3. Load the new map in the Editor going forward.

**Waiting on**: the Editor session to run step 1 (`ue_save_current.py`).

## Remaining tasks

- [ ] Save current spline edits as `YonginCircuitMap` (blocked on above)
- [ ] Resize/reposition the `floor` StaticMeshActor to cover the real track's
      extent (bbox ≈ 822m × 476m, centered ≈ (224, -78)m)
- [ ] Reposition `PlayerStart`, `StartFinishLine`, `Referee` to the new
      track's actual start point; confirm `Referee`'s `Cones` link still
      correctly references the populated spline actor
- [ ] Test drive in Play-in-Editor
- [ ] Package a standalone Linux build (`RunUAT.sh BuildCookRun`,
      command-line, no GUI needed) including the new map, so it can be
      launched day-to-day like the current `simulator/FSDS.sh` without
      needing the Editor open
- [ ] Verify via `fsds_ros2_bridge`: `/testing_only/track` shows **all**
      real cones (not a subset — this is exactly the kind of thing that
      silently broke before with `CustomMap`'s cap, so don't trust visuals
      alone), and the car can drive the real extent without falling off
      the world
- [ ] Update `SETUP_DEBUG_LOG.md` and `SIMULATOR_GUIDE.md`/`_KR.md` with the
      new from-source build process and how to launch the new map

## Useful scripts written so far (all in `~/`, run via
`exec(open("~/<name>").read())` in the Editor's Python (REPL) console)

- `ue_inspect_level.py`, `ue_list_actors.py` — level/actor inspection
- `ue_inspect_spline_cones.py`, `ue_inspect_spline_cones2.py` — spline_cones
  actor inspection
- `ue_populate_spline.py` — sets the real Yongin centerline onto the spline
  (already run, done)
- `ue_cleanup_and_frame.py` — removed the duplicate actor, selects the main
  spline for viewport framing (already run, done)
- `ue_save_current.py` — saves the current level (next step, not yet run)
- `/home/ailab/yongin_centerline.csv` — the real, recentered 256-point
  centerline data (meters) used to populate the spline
