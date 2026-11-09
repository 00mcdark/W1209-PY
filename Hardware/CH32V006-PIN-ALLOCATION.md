# Draft W1209 pin allocation — CH32V006F8P7 QSOP24

**Status: draft for schematic capture; not fabrication-ready.** This is a clean-sheet mapping for the retained W1209 interface and a direct, English-only port of W1209-PY. It is not a pin-compatible MCU swap. Verify all pin functions/electrical limits against the exact WCH datasheet and SDK before layout and prototype.

## MCU package pin map

| QSOP24 pin | MCU signal | Proposed board signal | Circuit notes |
|---:|---|---|---|
| 1 | PA1 | Display segment A | GPIO to segment driver/current-limit resistor |
| 2 | PA2 / ADC0 | NTC_SENSE | ADC input from probe/pull-up divider; keep analog route quiet |
| 3 | VSS | GND | Digital ground; connect to ground plane |
| 4 | PD4 / ADC7 | Display segment B | GPIO to segment driver/current-limit resistor |
| 5 | PD5 / ADC5 | Display segment C | GPIO to segment driver/current-limit resistor |
| 6 | PD6 / ADC6 | Display segment D | GPIO to segment driver/current-limit resistor |
| 7 | NRST | RESET | Expose test point/header; follow WCH reset circuit guidance |
| 8 | PD7 | Display segment E | GPIO to segment driver/current-limit resistor |
| 9 | PA4 | Spare GPIO / test pad | Keep unassigned initially; can be used after pin-mux review |
| 10 | VSSA | AGND | Analog ground; join to GND per WCH guidance near MCU/decoupling |
| 11 | VDDA | +5V_A | Feed from +5V through the datasheet-recommended analog supply arrangement; local bypass |
| 12 | PC0 | Display segment F | GPIO to segment driver/current-limit resistor |
| 13 | PC1 | Display segment G | GPIO to segment driver/current-limit resistor |
| 14 | PC2 | Display decimal point | GPIO to segment driver/current-limit resistor |
| 15 | PC3 | Display digit 1 enable | External digit driver; polarity depends on display common type |
| 16 | PC4 / ADC2 | Display digit 2 enable | GPIO to external digit driver; ADC function unused |
| 17 | PC5 | Display digit 3 enable | GPIO to external digit driver |
| 18 | PC6 | SET button | Button to GND, external pull-up to +5V recommended for defined reset behavior |
| 19 | PC7 | + button | Button to GND, external pull-up to +5V recommended |
| 20 | VDD | +5V | Main MCU supply; local bypass capacitor at pin |
| 21 | PD0 | - button | Button to GND, external pull-up to +5V recommended |
| 22 | PD1 / SWIO | WCH-Link SWIO | Reserve for single-wire debug/programming; expose on programming header |
| 23 | PD2 / ADC3 | RELAY_DRV | GPIO to relay transistor/MOSFET driver; external pull-down/base bias to ensure relay off at reset |
| 24 | PD3 / ADC4 | BUZZER_DRV | GPIO to buzzer transistor driver; do not drive a high-current buzzer directly |

### Signal groups

- **Display:** PA1, PD4, PD5, PD6, PD7, PC0, PC1, PC2 are segments A–G and DP; PC3–PC5 select the three digits.
- **Buttons:** PC6=SET, PC7=+, PD0=-. Use a consistent active-low circuit and confirm the selected GPIO/input configuration.
- **Temperature:** PA2/ADC0 is the NTC divider sense input. The R_PULLUP component must remain physically replaceable because P12 changes the assumed value in firmware but does not change the fitted resistor.
- **Relay:** PD2 controls an external driver; the coil uses the raw +12 V rail for a 12 V relay, preserving the usual W1209 supply/relay arrangement. Add a correctly rated flyback diode across the coil.
- **Sound:** PD3 drives an external transistor for an optional 5 V buzzer. Include an enable/disable provision if the buzzer circuit is omitted.
- **Programming:** PD1/SWIO, NRST, GND, and +5V target reference. Use the WCH-Link-compatible single-wire interface; it is not ARM SWD.

## External connectors and power

### J1 — 12 V input

- J1.1: +12V_IN
- J1.2: GND
- Proposed front end: input fuse or resettable fuse, reverse-polarity protection, and an appropriately rated transient clamp. Select based on the actual adapter, cable, and relay load.
- +12V_IN supplies the 12 V relay coil and feeds the logic regulator. Keep coil current return paths away from the NTC/ADC ground route.

### U2 — 12 V to 5 V logic regulator

- Buck regulator input: protected +12V_IN and GND.
- Output: +5V_LOGIC for MCU VDD, display, button pull-ups, and the optional 5 V buzzer circuit.
- Provide +5V_A for VDDA through the datasheet-appropriate filtering/decoupling arrangement. Tie VSSA to the analog-ground region and join analog/digital ground as directed by WCH; do not create an uncontrolled ground split.
- Choose regulator current/thermal ratings using worst-case display multiplex current, MCU current, buzzer current, and startup conditions. The 12 V relay coil is not powered from this 5 V regulator in this draft.
- Never connect +12V_IN directly to MCU, display logic, NTC divider, or programming-header target-voltage pins.

### J2 — NTC probe

- J2.1: NTC_SENSE → PA2/ADC0, with R_PULLUP from +5V_A to NTC_SENSE.
- J2.2: SENSOR_GND → AGND/GND return.
- Fit the pull-up footprint/value documented by the firmware profile (10 kΩ default); allow replacement if a different sensor range needs a different pull-up.
- Optional small input series resistor and capacitor footprint may be included, but finalize values by checking ADC acquisition requirements and measured noise. Keep the node away from relay contacts, coil switching, and display-current loops. Protect the ADC input from transients without compromising measurement accuracy.

### J3 — relay contacts

Expose COM/NO/NC (or the required subset) separately from the low-voltage logic connector. Size contacts, creepage/clearance, slots, fuse/protection, and terminal spacing for the intended load and applicable safety requirements. Do not infer a safe mains rating from the W1209 form factor.

### J4 — programming/debug

| Header pad | Signal |
|---:|---|
| 1 | +5V target reference (do not use as a programmer power output unless supported by the selected tool) |
| 2 | GND |
| 3 | PD1 / SWIO |
| 4 | NRST |

Confirm header orientation and target-voltage behavior with the exact WCH-Link model. Leave PD1 free of heavy loads and ensure board circuitry cannot prevent programming.

## Driver and display implementation notes

- Identify the actual 3-digit display part and whether it is common-anode or common-cathode before choosing digit-driver polarity. Use eight segment current-limit resistors (or a suitably rated resistor network) and external digit/segment transistors where needed. Do not exceed MCU pin or total-port current ratings.
- Use a base/gate resistor and a defined reset-state bias on the relay switch. Select the relay transistor/MOSFET and flyback diode for the coil current and 12 V transients. The MCU must default to relay OFF during reset, startup, brownout, or firmware fault.
- Use a transistor driver for the buzzer if its current exceeds the verified GPIO rating. A passive piezo may need a timer/PWM output; this draft's PD3 assignment requires confirming peripheral remap or implementing suitable software drive.
- Use low-voltage and dummy-load bring-up first. Verify display polarity, segment current, ADC transfer curve, relay default state, coil suppression, temperature fault detection, and timer/alarm behavior before connecting an operational load.

## Pin-mux and implementation checks before release

1. Verify this exact package pin-numbering and every alternate function against the latest WCH CH32V006 datasheet and reference manual.
2. Confirm ADC0 is available on PA2 and configure PA2 analog; check ADC reference range, sample time, calibration, oversampling/filtering, and input impedance against WCH documentation.
3. Confirm PD1 remains available for WCH-Link SWIO with selected option bytes and startup configuration; do not reuse PY32 SWD assumptions.
4. Check whether PA4/PD7 remapping constraints apply to the selected function configuration. PA4 is intentionally spare here; do not enable a conflicting alternate function.
5. Confirm all GPIO voltage/current limits at +5V VDD. Use external drivers for digit commons, relay coil, and buzzer.
6. Confirm the CH32V006 flash geometry, erase/program operation, endurance and protected/reserved regions before implementing five-profile storage. Do not reuse PY32 flash addresses or routines.
7. Verify the board outline, display footprint, terminal positions, mounting holes, and relay insulation spacing against the intended enclosure and actual hardware.

## References

- Pinout reference supplied for this project: https://chiptron.cz/pinouty/ch32v006f8p7-pinout/
- WCH CH32V006 community documentation/SDK: https://github.com/ch32-riscv-ug/CH32V006
- WCH CH32V006 datasheet: https://www.wch-ic.com/downloads/CH32V006DS0_PDF.html
- WCH CH32V00x reference manual: https://www.wch.cn/downloads/CH32V00XRM_PDF.html
- Existing behavior reference: https://github.com/Saco-de-patatas/W1209-PY/blob/main/README.EN.MD

This draft is a pin-level starting point for schematic capture, not a released schematic, PCB, or electrical-safety certification.
