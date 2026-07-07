# Changelog

## 2026-07-07

Macro cleanup: removed unused macros, moved rare calibrations into pluggable `.disabled.cfg` modules.

- Removed macros:
  - `BED_SCREWS_MANUAL` (broken: calls `BED_SCREWS_ADJUST` but no `[bed_screws]` section exists).
  - `VERIFY_BED_LEVEL` (duplicate of `LEVEL_BED_SCREWS`).
  - `BED_LEVELING_QUICK`, `LOAD_MESH`, `CLEAR_MESH` (trivial wrappers; the tap workflow builds a mesh every print, Fluidd has UI for profiles).
  - `PID_Quick` (trivial composition of `PID_Hotend` + `PID_Bed`).
  - `EMERGENCY_STOP`, `RESTART_KLIPPER`, `MOTORS_OFF`, `HEATERS_OFF`, `FANS_OFF` (all covered by built-in Fluidd controls; the real e-stop is the Fluidd M112 button). `ALL_OFF` now calls `_SYSTEMS_OFF` directly.
- New pluggable modules (rename to `.enabled.cfg` to use):
  - `extension/eddy_calibration.disabled.cfg`: one-time Eddy setup macros moved out of `btt_eddy_duo.enabled.cfg` (`EDDY_CALIBRATE_CURRENT`, `EDDY_CALIBRATE_PROBE`, `EDDY_CALIBRATE_DRIFT`/`_NEXT`/`_STOP`, `EDDY_CALIBRATE_ALL`). `EDDY_CALIBRATE_TAP` stays enabled (needed on every nozzle change).
  - `extension/leveling_corners.disabled.cfg`: paper-test corner leveling (`LEVEL_CORNERS`/`LEVEL_CORNERS_NEXT`) moved out of `leveling_manual.enabled.cfg` as an Eddy-less fallback.
  - `extension/pid.enabled.cfg` renamed to `pid.disabled.cfg` (PID calibration is only needed after hardware changes).
- Deleted `extension/btt_eddy_duo_ng.disabled.cfg` (obsolete Kalico/eddy-ng draft, superseded by native tap).
- Slicer-facing macros verified against the OrcaSlicer profiles and kept: `START_PRINT`, `END_PRINT`, `FAN_LAYER_CHECK`, `M600`.
- Manual follow-up in Fluidd (Settings -> Macros): hide the override/compat macros (`PAUSE`, `RESUME`, `CANCEL_PRINT`, `BED_MESH_CALIBRATE`, `Z_OFFSET_APPLY_PROBE`, `M17`, `M109`, `M190`, `M420`, `M600`, `FAN_LAYER_CHECK`) — Fluidd only auto-hides `_`-prefixed names.

## 2026-07-05

- Switched the Eddy Z-reference workflow from scan-based probing to native Klipper "tap" probing (nozzle contact, requires Klipper >= v0.13.0-700).
- `extension/btt_eddy_duo.enabled.cfg`:
  - Removed the `G28` override, `SET_Z_FROM_PROBE`, `RESTORE_PROBE_OFFSET`, and the `SET_GCODE_OFFSET` runtime-offset machinery (superseded by tap).
  - Added `EDDY_TAP` / `_EDDY_TAP_APPLY`: tap probe at the bed center, then `SET_KINEMATIC_POSITION` makes the contact point the true Z=0.
  - Added `EDDY_CALIBRATE_TAP STAGE=guess|refine|verify` wrapper around `PROBE_EDDY_CURRENT_TAP_CALIBRATE`.
  - `Z_OFFSET_APPLY_PROBE` now folds the current Z G-Code offset into `tap_z_offset` (`METHOD=tap`).
  - `BED_MESH_CALIBRATE` default `METHOD` changed from `rapid_scan` to `scan` (better accuracy per Klipper docs).
  - `EDDY_STATUS` reports `tap_threshold` / `tap_z_offset` instead of the removed runtime offset.
- `extension/start_end.enabled.cfg`: `START_PRINT` caps the extruder at 170C for the tap phase (part fan assists cooldown on hot restarts), wipes the nozzle without purging, runs `EDDY_TAP`, scans the bed mesh, then heats to full printing temperature.
- `extension/nozzle.enabled.cfg`: `CLEAN_NOZZLE_BRUSH` accepts `PURGE=0` and homes via `_HOME_IF_NEEDED` instead of an unconditional `G28`.
- `extension/leveling_auto.enabled.cfg`: `BED_LEVELING` full mesh uses `METHOD=scan`.
- Required on-printer calibration before first use: `EDDY_CALIBRATE_TAP STAGE=guess`, then `refine`, then `verify`, then `SAVE_CONFIG`.
- The `nvm_offset` saved variable is no longer used and may be removed from `variables.cfg`.

Macro audit fixes (same day, after live testing):

- `extension/leveling_auto.enabled.cfg`:
  - `BED_LEVELING` now runs the tap workflow: tap-safe extruder temperature (capped at 170C), brush wipe without purge, `EDDY_TAP` (skippable with `TAP=0`), then a full `METHOD=scan` mesh; the pre-mesh filament purge was removed (it left goop on the nozzle right before probing).
  - `MESH_INFO` rewritten: `printer.bed_mesh` has no `average`/`std_dev`/`mesh_x_count` fields and `mesh_min`/`mesh_max` are XY pairs (this crashed with "tuple object has no element 2"); stats are now computed from `probed_matrix`.
  - `LOAD_MESH` reported stats evaluated before the profile actually loaded; it now delegates to `MESH_INFO` after loading.
- `extension/leveling_manual.enabled.cfg`:
  - `LEVEL_CORNERS` rewritten as a state machine (`LEVEL_CORNERS` -> `LEVEL_CORNERS_NEXT`): `PAUSE` inside a macro cannot stop macro expansion in Klipper, so the old version drove through all corners at once; it also used hardcoded positions with Y=-12.5 outside the reachable range ("Move out of range"). Corner positions are now derived from `screws_tilt_adjust` coordinates plus probe offsets, clamped to axis limits.
  - Removed `LEVEL_BED_SCREWS_LOOP`: same `PAUSE`-inside-macro flaw made it probe five times back to back with no chance to adjust.
  - Header comment updated to the real probe offsets and coordinate model.
- `extension/system_control.enabled.cfg`: removed the nonexistent `M410` from `EMERGENCY_STOP`; fixed mojibake in `SHUTDOWN` messages.
- `extension/mcode_compat.enabled.cfg`: fixed mojibake in `M109`/`M190` messages.
- `extension/btt_eddy_duo.enabled.cfg`: drift calibration messages now state 60C to match `TEMPERATURE_PROBE_CALIBRATE TARGET=60`.
- Note: `PROBE_CALIBRATE` does not exist for `probe_eddy_current`; use `EDDY_CALIBRATE_PROBE` (descend calibration) or `EDDY_CALIBRATE_TAP`.

Expected-behavior alignment (same day):

- `EDDY_TAP` now takes multiple tap samples and averages them (`SAMPLES=3` by default); reports per-sample contact Z, average, and spread, and warns when the spread exceeds 0.025mm. Implemented via `_EDDY_TAP_COLLECT` accumulator (macro templates expand before execution, so results must be gathered by a separate macro after each `PROBE`).
- Bed screw adjustment now uses tap probing: `LEVEL_BED_SCREWS` and `VERIFY_BED_LEVEL` run `SCREWS_TILT_CALCULATE METHOD=tap`. With `METHOD=tap` Klipper measures at the NOZZLE (probe XY offsets are zeroed), so `[screws_tilt_adjust]` coordinates are nozzle positions near the physical screws. A bare `SCREWS_TILT_CALCULATE` without `METHOD=tap` must not be used with these coordinates.
- `[screws_tilt_adjust]` coordinates corrected per owner-confirmed hardware: physical screws form a ~170x170mm square centered on the bed - (30,30), (200,30), (200,200), (30,200); the previous sensor-based values implied a wrong 120x120 layout. Tap points are pulled inward where the Eddy sensor would otherwise leave the bed (+15mm in X at the left screws, -5mm in Y at the rear ones): rear_left 45,195; rear_right 200,195; front_right 200,30; front_left 45,30. The inward offset does not bias leveling: at convergence all points read equal, so a flat bed ends up level.
- `LEVEL_CORNERS` uses the same physical coordinates directly (no offset math).
- Adaptive mesh minimum point count: no config change needed - mainline `bed_mesh.set_adaptive_mesh` clamps the adapted probe count to at least 3 (or 4) per axis, so an adaptive scan always has >= 9 points; verified against klippy/extras/bed_mesh.py.
- Z-offset adjustment via tap was already in place: `Z_OFFSET_APPLY_PROBE` maps to `Z_OFFSET_APPLY_PROBE_ORIG METHOD=tap`, which updates `tap_z_offset` (verified against klippy/extras/probe_eddy_current.py `_save_tap_z_offset`).

Bed mesh switched from sensor scanning to tap probing (same day):

- `BED_MESH_CALIBRATE` (override) default `METHOD` is now `tap`: every mesh point is measured by nozzle contact instead of a sensor reading. Applies to both the adaptive mesh in `START_PRINT` and the full mesh in `BED_LEVELING` (`METHOD=tap ADAPTIVE=0`).
- The override selects the start height per method: `HORIZONTAL_MOVE_Z=5` for tap (tap needs a 3-20mm start; starting at 1mm risks an undetected contact and a crash) and `HORIZONTAL_MOVE_Z=1` for scan/rapid_scan. `[bed_mesh] horizontal_move_z` default changed 1.0 -> 5.0 accordingly.
- The override refuses `METHOD=tap` with an uncalibrated `tap_threshold`.
- `[bed_mesh]` bounds changed from 15,26..195,215 to 45,26..195,200: with tap the mesh points are NOZZLE positions and the Eddy sensor (offset -34.20/+24.15) must stay above the bed at every point (x >= 45, y <= 200); x <= 195 keeps the area valid for scan as well. Trade-off: the leftmost ~45mm and rear ~30mm of the bed are outside the mesh; Klipper extends edge values for prints there.

## 2026-04-13

- Updated the project to Klipper `v0.13.0-623`.
- Updated Eddy configuration for compatibility with Klipper `v0.13.0-623`.
- Replaced deprecated `z_offset` usage in `probe_eddy_current` context with `descend_z`.
- Replaced deprecated `printer.probe.last_z_result` usage with `printer.probe.last_probe_position`.
- Renamed local template variable `z_offset` to `final_probe_z` to avoid deprecated name detection in macros.
- Updated user-facing macro messages to reflect the new meaning:
  - `Z Offset Calibration` renamed to `Descend Z Calibration`.
  - `Runtime Z Offset` renamed to `Runtime Probe Adjust`.
  - `result Z Offset` renamed to `Applied Probe Z`.
  - `Last Probe Result` renamed to `Last Probe Z`.
- Updated files:
  - `extension/btt_eddy_duo.enabled.cfg`
  - `extension/btt_eddy_duo_ng.disabled.cfg`
