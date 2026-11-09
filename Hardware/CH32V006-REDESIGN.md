# W1209 CH32V006F8P7 clean-sheet port

## Scope

This branch is for a new W1209-format controller built around the WCH **CH32V006F8P7 (QSOP24)**. Preserve the user-facing W1209 interface: 12 V input, relay output, NTC probe connection, three-digit 7-segment display, and SET / + / - buttons. Add the optional buzzer/alarm circuit. The firmware target is a **direct port of the existing W1209-PY behavior**, with English strings only.

This is not a PY32 pin-compatible swap. The existing firmware's PY32 clock, GPIO, ADC, interrupt, flash, startup, and build code must be replaced/adapted for CH32V006. Reuse the thermostat behavior and menus rather than the old MCU's pin map or register drivers.

## Feature scope

Port all functional options in the reference firmware, excluding Spanish and the language selector:

- P0 operating mode (heater, cooler, active-inside-window, active-outside-window)
- P1 hysteresis/window width; P2/P3 bounds; P4 offset; P5 delay units
- P6/P7 independent high/low alarms
- P8 Celsius/Fahrenheit; P9 defaults/profile erase
- P10 timer (up to 999 days); P11 target digit editing
- P12 pull-up value; P13/P14 R0/T0; P15 beta; P16/P17 R1/T1
- P18/P19 sensor-resistance snapshots; P20 cable resistance
- P21/P22 relay-on/off delay; P23 filter depth; P24 ADC effective depth; P25 ADC offset
- P26 scrolling temperature; P27 explanatory messages; P28 sound; P29 five configuration profiles
- Manual relay override, sensor fault indication, and English rolling messages

P30 language selection is omitted because English is fixed. Keep all other supported behavior. The exact menu labels can remain the existing English 7-segment strings.

## Initial QSOP24 pin allocation proposal

The following is a **logical first-pass allocation**, not a released schematic. It is based on the CH32V006 QSOP24 pin assignment in the WCH datasheet. Confirm the selected display's common polarity/current, the exact ADC channel names, pin alternate-function constraints, and electrical limits against the exact datasheet revision and prototype before layout release.

| MCU pin | Proposed signal | Notes |
| --- | --- | --- |
| PA1 | Display segment A | GPIO |
| PA2 / ADC0 | NTC divider sense | Keep analog routing quiet; ADC input must stay within VDDA/VSSA |
| PD4 / ADC7 | Display segment B | GPIO use |
| PD5 / ADC5 | Display segment C | GPIO use |
| PD6 / ADC6 | Display segment D | GPIO use |
| PD7 | Display segment E | Avoid assigning PA4 simultaneously if package multiplexing constrains them |
| PC0 | Display segment F | GPIO |
| PC1 | Display segment G | GPIO |
| PC2 | Display decimal point | GPIO |
| PC3 | Digit select 1 | Transistor driver as required by display common polarity |
| PC4 / ADC2 | Digit select 2 | GPIO use |
| PC5 | Digit select 3 | GPIO use |
| PC6 | SET button | Input with external/pull-up strategy verified for supply and reset states |
| PC7 | + button | Input |
| PD0 | - button | Input |
| PD2 / ADC3 | Relay driver | Low-side transistor/MOSFET stage; default relay-off at reset |
| PD3 / ADC4 | Optional buzzer driver | Transistor driver if buzzer current exceeds GPIO rating |
| PD1 | WCH-Link SWIO | Reserve for programming/debug; expose test pad/header |
| PA4 | Spare / optional buzzer alternative | PA4/PD7 alternate-function relationship to be checked before using both |
| NRST | Reset | Expose test pad; follow WCH reset recommendations |
| VDD, VSS | Digital power and ground | Local bypass capacitor close to MCU |
| VDDA, VSSA | Analog supply and ground | Follow datasheet decoupling/layout recommendations; connect per datasheet |

The allocation uses one ADC-capable input for the probe, one GPIO per segment and digit-select line, three buttons, relay, buzzer, and debug. Do not treat unused ADC alternate functions as unavailable GPIOs. Final pin mux setup must explicitly disable conflicting peripherals and preserve SWIO.

## Hardware design constraints

- Regulate the 12 V input to a rail valid for the MCU and display. Do not feed 12 V to MCU pins. Select the regulator and thermal/power components for the relay coil and buzzer load.
- Keep the NTC divider and ADC node away from relay contacts, coil switching, and high-current return paths. Add only datasheet- and measurement-validated input filtering.
- Retain the replaceable sensor pull-up resistor (reference design P12) and document its fitted value. Firmware configuration does not replace physically changing this resistor.
- Drive the relay coil through a correctly rated transistor/MOSFET and flyback suppression; use safe reset defaults. Keep relay contact isolation/clearances suitable for the intended voltage/current and applicable safety rules. Do not infer mains ratings from the original module.
- Drive display commons/segments through suitable current-limiting and switching components. Verify multiplex duty cycle and peak current with the actual display part.
- Bring out SWIO, NRST, GND, and a suitable target supply reference for WCH-Link programming/debugging.
- Physical outline, terminal positions, mounting holes, relay/contact spacing, and display footprint still need verification against the intended enclosure and actual W1209 unit.

## Firmware port plan

1. Keep the existing application logic, data model, temperature conversions, menu semantics, filtering, alarms, timer, profiles, and English message table as the behavioral baseline.
2. Separate board-specific code behind a small hardware interface: tick/timebase, GPIO/button read, 7-segment scan, ADC sample, relay/buzzer control, and nonvolatile profile storage.
3. Replace PY32-specific startup, register headers, clock/GPIO/ADC/interrupt initialization, CRC and flash routines with WCH CH32V006 startup/SDK equivalents. Build with the WCH RISC-V toolchain and WCH-Link-compatible programming flow.
4. Remove Spanish message table, language state, P30 menu, and language persistence; force English on loading defaults and profiles. Version and validate stored profiles so firmware upgrades cannot interpret stale layouts unsafely.
5. Implement a bounded, wear-conscious profile journal using the exact CH32V006 flash geometry and erase/program constraints. Do not reuse the PY32 page addresses or routines.
6. Validate first in host-side tests for temperature conversion, heating/cooling/window decisions, alarm boundaries, timer countdown, profile CRC/version handling, and digit editing. Then verify on the board with a current-limited supply and dummy relay load before any mains-connected use.

## Source references

- Existing behavior and English menu descriptions: [W1209-PY README.EN.MD](https://github.com/Saco-de-patatas/W1209-PY/blob/main/README.EN.MD)
- Existing firmware baseline: [this repository's User/main.c](../User/main.c)
- CH32V006 datasheet (WCH): https://www.wch-ic.com/downloads/CH32V006DS0_PDF.html
- CH32V00x reference manual (WCH): https://www.wch.cn/downloads/CH32V00XRM_PDF.html

The published pin allocation is provisional until verified directly against the WCH package pin table and board prototype. This document is a design baseline, not a fabrication-ready schematic/PCB or verified firmware build.
