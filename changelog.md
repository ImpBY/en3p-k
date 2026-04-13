# Changelog

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
