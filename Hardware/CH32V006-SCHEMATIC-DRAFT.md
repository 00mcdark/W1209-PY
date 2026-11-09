# W1209 CH32V006F8P7 schematic-first draft

**Revision:** 0.1 — functional connectivity draft, not fabrication-ready.  
**Target:** CH32V006F8P7 QSOP24, W1209-style 12 V input, three-digit display, SET/+/- buttons, NTC probe, 12 V relay, optional buzzer, WCH-Link SWIO programming.  
**Firmware intent:** direct W1209-PY behavior port, English only.

This draft specifies connections and protection blocks so a KiCad schematic can be captured. It deliberately does not claim final regulator compensation/layout, display-current compliance, relay-contact safety rating, or CH32 pin electrical-limit validation. Verify each component against its datasheet and the exact WCH MCU datasheet before PCB release.

## 1. Top-level sheet connectivity

```text
J1 12V_INPUT -> F1 -> reverse-polarity protection -> +12V_PROT
                                      |                    |
                                      |                    +--> K1 12V relay coil --> Q1 low-side switch --> GND
                                      |                                      D1 flyback diode across coil
                                      +--> U1 5V buck --> +5V_LOGIC --> MCU VDD, display, buttons, optional buzzer
                                                           +--> filtered +5V_A --> MCU VDDA and NTC pull-up

J2 NTC: +5V_A -- R_PULLUP (replaceable) --+-- NTC_SENSE --> PA2/ADC0
                                           |
                                          NTC
                                           |
                                          AGND

MCU GPIOs --> display segment resistors/driver and three digit drivers
Buttons: MCU inputs pulled up to +5V_LOGIC; switches connect inputs to GND
WCH-Link header: +5V target reference, GND, PD1/SWIO, NRST
```

## 2. MCU symbol and package pin connections

Use the exact CH32V006F8P7 QSOP24 pin numbering from the datasheet/package footprint. The proposed signal assignment is:

| MCU package pin | Net | Function |
|---:|---|---|
| 1 PA1 | SEG_A | Segment A |
| 2 PA2 / ADC0 | NTC_SENSE | NTC ADC input |
| 3 VSS | GND | Digital ground |
| 4 PD4 / ADC7 | SEG_B | Segment B |
| 5 PD5 / ADC5 | SEG_C | Segment C |
| 6 PD6 / ADC6 | SEG_D | Segment D |
| 7 NRST | NRST | Reset/programming header |
| 8 PD7 | SEG_E | Segment E |
| 9 PA4 | GPIO_SPARE | Leave unused in rev 0.1 |
| 10 VSSA | AGND | Analog ground, joined as specified below |
| 11 VDDA | +5V_A | Filtered/decoupled analog supply |
| 12 PC0 | SEG_F | Segment F |
| 13 PC1 | SEG_G | Segment G |
| 14 PC2 | SEG_DP | Decimal point |
| 15 PC3 | DIGIT_1 | Digit driver control |
| 16 PC4 / ADC2 | DIGIT_2 | Digit driver control |
| 17 PC5 | DIGIT_3 | Digit driver control |
| 18 PC6 | BTN_SET_N | SET, active low |
| 19 PC7 | BTN_PLUS_N | +, active low |
| 20 VDD | +5V_LOGIC | Main supply |
| 21 PD0 | BTN_MINUS_N | -, active low |
| 22 PD1 / SWIO | SWIO | WCH-Link single-wire debug/programming |
| 23 PD2 / ADC3 | RELAY_GATE | Relay low-side switch control |
| 24 PD3 / ADC4 | BUZZER_CTRL | Optional buzzer control |

Add a no-connect marker only to PA4 if it remains unused; do not omit required VDD/VSS/VDDA/VSSA connections. Verify that the exact MCU variant and toolchain use PD1 as SWIO with the selected option-byte configuration.

## 3. Power input and 5 V regulator sheet

### J1, protection, and input rail

- J1.1 = `VIN_12V`; J1.2 = `GND`.
- F1 in series with J1.1; select its rating for the intended adapter, relay coil current, regulator input current, and wiring.
- Add reverse-polarity protection in series (Schottky diode or correctly rated P-MOSFET ideal-diode arrangement). Select voltage/current/surge ratings for the actual supply.
- Add an optional TVS `DTVS` from `+12V_PROT` to GND, chosen for the adapter's maximum voltage and expected transients. Do not select solely by nominal 12 V label.
- Place bulk input capacitor `CIN` from `+12V_PROT` to GND near U1 (starting footprint 47 uF, voltage rating >=25 V) and a 100 nF ceramic close to the regulator input. Validate ripple and surge conditions.
- `+12V_PROT` feeds only the relay coil and U1 buck input in this draft; do not route it to MCU pins or the programming header.

### U1 buck regulator functional connections

Use a 12 V-input, fixed 5 V-output buck regulator with a datasheet reference design rated for the board's worst-case load. The schematic must include its required switching components and correct layout recommendations; **do not substitute an unverified random inductor/diode/compensation network**.

- U1 VIN = `+12V_PROT`; GND = power ground.
- U1 SW -> L1 -> `+5V_LOGIC` (or use the exact topology in the selected regulator's datasheet).
- For an asynchronous regulator, connect its specified Schottky catch diode from GND to SW with polarity per the reference design; omit only if the selected IC integrates the rectifier.
- U1 FB senses `+5V_LOGIC` at the output capacitor/load per the datasheet. If fixed 5 V, strap/configure as the datasheet says.
- `COUT` from +5V_LOGIC to GND; choose capacitance, ESR, and voltage rating exactly as required by the regulator. Add 10 uF bulk plus local 100 nF near MCU/display power entry as an initial downstream footprint set.
- EN/BOOT/COMP/SS pins (if present) wired exactly as the selected regulator data sheet reference circuit specifies.
- Name regulator outputs `+5V_LOGIC`. Select a part with enough margin for display peak current, MCU, button pull-ups, and buzzer. Relay coil remains on `+12V_PROT`.

### Analog supply and grounds

- `+5V_LOGIC` -> ferrite bead or small resistor `FB1` -> `+5V_A`; add 1 uF and 100 nF at VDDA/analog region. Final filter choice must follow WCH ADC supply guidance and avoid excessive supply drop.
- MCU VDD pin 20: `+5V_LOGIC`, with 100 nF ceramic directly between pin 20 and the nearest GND return. Add another 1 uF local ceramic nearby.
- MCU VDDA pin 11: `+5V_A`, with local 100 nF and 1 uF to VSSA, close to the pins.
- MCU VSS pin 3: GND; MCU VSSA pin 10: AGND. Join AGND to GND at a single quiet point near the MCU/ADC return, following WCH recommendations. Keep display/relay current pulses out of the NTC return path.
- Provide at least one 100 nF bypass per supply pin pair and keep loops short. Confirm capacitor values and placement with the MCU and buck datasheets.

## 4. NTC connector and divider

- J2.1 = `NTC_SENSE`; J2.2 = `AGND`.
- R_PULLUP from `+5V_A` to `NTC_SENSE`; make this resistor replaceable (through-hole or accessible resistor footprint). Initial fitted value: 10 kOhm, subject to thermistor and range validation.
- Thermistor connects between `NTC_SENSE` and AGND. For the common 10 kOhm NTC this gives high resistance near room temperature and falling resistance as temperature rises.
- PA2/ADC0 senses `NTC_SENSE`. Provide optional C_NTC from NTC_SENSE to AGND (initial footprint 10 nF; populate only after ADC acquisition/noise testing) and optional small series resistor footprint between connector/divider and ADC pin. Choose final values to satisfy ADC sampling settling and response requirements.
- Add optional ESD/overvoltage protection footprints for the probe connector only if leakage/capacitance are suitable for the measurement range; validate accuracy before populating.
- Sensor wiring is low voltage only. Do not connect any external voltage source to the probe input.

## 5. Buttons

Each button uses a separate active-low input:

- `+5V_LOGIC` -- 10 kOhm pull-up `R_SET` -- `BTN_SET_N` (PC6); `BTN_SET_N` -- SW_SET -- GND.
- `+5V_LOGIC` -- 10 kOhm pull-up `R_PLUS` -- `BTN_PLUS_N` (PC7); `BTN_PLUS_N` -- SW_PLUS -- GND.
- `+5V_LOGIC` -- 10 kOhm pull-up `R_MINUS` -- `BTN_MINUS_N` (PD0); `BTN_MINUS_N` -- SW_MINUS -- GND.
- Optional 1 nF–10 nF debounce capacitor footprint from each button input to GND; firmware debounce remains required. Confirm whether GPIO internal pulls are enabled/disabled in firmware so external and internal biasing do not conflict.
- Button switch wiring and pull-ups must remain within the MCU pin voltage limits at all power/reset conditions.

## 6. Relay driver

Use Q1 as a low-side N-channel MOSFET switch sized for the selected 12 V coil. This prevents coil current from flowing through the MCU pin.

- K1 coil high side -> `+12V_PROT`; coil low side -> Q1 drain; Q1 source -> GND.
- Q1 gate <- `RELAY_GATE` (PD2) through R_GATE (100 Ohm initial footprint); R_GS (100 kOhm) from gate to source/GND ensures OFF during reset.
- D1 flyback diode directly across K1 coil: cathode to `+12V_PROT`, anode to Q1 drain/coil low side. Select diode for coil current and release-time needs. If faster release is required, use a properly engineered clamp instead of changing diode polarity.
- Select Q1 for logic-level gate drive at the MCU's actual output voltage, coil current, VDS transients, and dissipation. Verify gate threshold is not mistaken for a full-on specification.
- Keep coil driver loop compact and separate from the NTC/ADC area. Firmware initializes PD2 inactive before normal operation.
- Route relay contacts to separate terminals J3 (COM/NO/NC as required). Contact creepage, clearance, fuse, slot, terminal rating, enclosure, and isolation must be designed for the actual load and applicable safety standards. This low-voltage control schematic does not certify mains switching safety.

## 7. Three-digit 7-segment display

### MCU control nets

- SEG_A=PA1, SEG_B=PD4, SEG_C=PD5, SEG_D=PD6, SEG_E=PD7, SEG_F=PC0, SEG_G=PC1, SEG_DP=PC2.
- DIGIT_1=PC3, DIGIT_2=PC4, DIGIT_3=PC5.
- Use the existing display part only after identifying its exact part number, common polarity, pinout, forward voltage, and allowed peak current.

### Current limit and digit selection

- One resistor per segment net, between the MCU segment driver and display segment bus. Do not share one resistor for an entire digit. Starting resistor footprint: 1 kOhm per segment to limit peak current conservatively; calculate and adjust for the chosen display, supply, driver drops, desired brightness, and MCU limits. Confirm average current at the chosen multiplex duty cycle.
- Digit common switching depends on display polarity:
  - **Common-anode display:** one PNP high-side transistor per digit (or appropriate P-channel device), emitters/source to +5V_LOGIC, collectors/drains to the three digit common pins; base/gate resistors and off-state pull resistors. Segment drivers must sink current and be compatible with MCU output levels/current.
  - **Common-cathode display:** one NPN low-side transistor per digit (or appropriate N-channel device), emitters/source to GND, collectors/drains to the digit common pins; base/gate resistors and off-state pull resistors. Segment drivers must source current and be compatible with MCU output levels/current.
- Add transistor driver stages to segment lines if the required per-pin/aggregate current exceeds MCU limits. Verify the whole port's current limits, not only individual pin ratings.
- Make digit enables default OFF at reset with hardware bias resistors. Multiplex firmware should blank digits before changing segment data to avoid ghosting.
- Confirm 5 V GPIO tolerance/output-high capability for this MCU from the datasheet before directly connecting logic nets. If not compatible, add level shifting/drivers.

## 8. Optional buzzer

- `BUZZER_CTRL` (PD3) -> R_B (1 kOhm initial footprint) -> base of Q2 NPN; emitter -> GND; collector -> buzzer negative. Buzzer positive -> its rated supply rail.
- R_BPD (100 kOhm) base-to-GND keeps Q2 off during reset.
- For an inductive/magnetic buzzer, add a flyback diode across the buzzer with cathode to positive supply. For a piezo buzzer, do not add a flyback diode unless its datasheet specifies one.
- Verify the buzzer supply/current and whether the firmware needs PWM. Omit Q2/buzzer or leave DNP if acoustic alarm is not fitted; retain visual alarm behavior.

## 9. WCH-Link programming header J4

| J4 pin | Net | Connection |
|---:|---|---|
| 1 | +5V_TARGET | Reference only; do not back-power board unless programmer documentation explicitly allows it |
| 2 | GND | Common ground |
| 3 | SWIO | PD1, MCU package pin 22 |
| 4 | NRST | MCU package pin 7 |

Keep SWIO free of capacitive/low-impedance loads. Confirm header orientation and voltage reference requirements against the exact WCH-Link model. Do not label this interface ARM SWD.

## 10. Draft BOM/function list

| Ref | Part/function | Preliminary requirement |
|---|---|---|
| U1 | 12 V to 5 V buck regulator | Select from manufacturer datasheet reference design, adequate current and thermal margin |
| L1, D2, CIN, COUT | Buck power components | Values/topology depend on U1; use recommended layout and parts |
| F1 | Input fuse | Size for adapter, relay coil, and logic load |
| DREV | Reverse-polarity protection | Voltage/current/surge rating for 12 V input |
| DTVS | Input transient suppressor | Select from actual supply and transient environment |
| FB1, CVDDA | VDDA filter/decoupling | Follow WCH analog supply recommendations |
| R_PULLUP | NTC pull-up | Replaceable, initial 10 kOhm |
| Q1, D1 | Relay MOSFET + flyback diode | Sized to actual relay coil current/voltage |
| Q2 | Optional buzzer transistor | Sized to buzzer voltage/current |
| R_SEG_A..DP | Segment current limits | One per segment, calculate for display and drivers |
| QD1..QD3 | Digit drivers | Polarity/package based on display common type |
| R_SET/R_PLUS/R_MINUS | Button pull-ups | Initial 10 kOhm each |
| J1/J2/J3/J4 | Power/probe/relay/programming connectors | Voltage/current/contact spacing appropriate to application |

## 11. Bring-up checklist

1. Check schematic ERC and confirm all 24 package pins, power pins, and footprint numbering.
2. Validate regulator output/current/thermal behavior with a dummy load before installing the MCU.
3. Power at current-limited 12 V with relay contacts unloaded; confirm +5V_LOGIC, +5V_A, ripple, VDDA/VSSA levels, and no 12 V on logic pins.
4. Test WCH-Link SWIO and NRST before application firmware is loaded.
5. Test display scan using conservative current and verify digit-driver polarity and reset-off state.
6. Measure the NTC divider/ADC curve with known resistors and a temperature reference; check noise when display and relay drivers switch.
7. Verify relay Q1 off at reset, correct coil current, and flyback suppression with contacts unloaded.
8. Test buttons, buzzer, timers, alarms, profiles, and sensor faults in firmware; then test relay control with a safe low-voltage dummy load.
9. Have relay contact spacing, isolation, fusing, enclosure, and intended-load safety reviewed before any mains-connected use.

## References

- CH32V006F8P7 pinout reference: https://chiptron.cz/pinouty/ch32v006f8p7-pinout/
- CH32V006 documentation/SDK: https://github.com/ch32-riscv-ug/CH32V006
- WCH CH32V006 datasheet: https://www.wch-ic.com/downloads/CH32V006DS0_PDF.html
- WCH CH32V00x reference manual: https://www.wch.cn/downloads/CH32V00XRM_PDF.html
- W1209-PY English behavior reference: https://github.com/Saco-de-patatas/W1209-PY/blob/main/README.EN.MD

**This is a schematic-first connectivity draft. It is not a completed KiCad schematic, fabrication-ready PCB, validated firmware build, or safety certification.**
