# CH32V006 firmware port status

Target: W1209-format thermostat, CH32V006F8P7 QSOP24, English-only direct port of W1209-PY behavior.

- [x] Confirm target MCU and feature scope.
- [x] Create clean-sheet working branch: `ch32v006-clean-sheet`.
- [x] Record a provisional QSOP24 pin-allocation and safety constraints in [CH32V006-REDESIGN.md](Hardware/CH32V006-REDESIGN.md).
- [ ] Verify MCU package pin mux, ADC input mapping, reset/debug, flash geometry and limits against WCH datasheet/reference manual.
- [ ] Create and review CH32V006 schematic and PCB.
- [ ] Add CH32V006 toolchain/startup/build target.
- [ ] Port timebase, display scan, buttons, ADC, relay, buzzer, CRC and profile storage.
- [ ] Remove Spanish support and language menu while preserving all other options.
- [ ] Add host tests and run target build.
- [ ] Bench-validate sensing, display, relay, alarms, timing and persistence before deployment.

No board files or target firmware build are claimed complete at this stage. In particular, the pin allocation is a proposal, and the 12 V input / relay-contact safety details must be resolved from the actual board constraints and intended load.
