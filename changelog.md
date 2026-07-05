# Changelog

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
