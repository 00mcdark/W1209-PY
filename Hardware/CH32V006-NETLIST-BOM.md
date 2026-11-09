# W1209 CH32V006F8P7 schematic netlist and BOM proposal

**Status: Rev A design proposal for schematic capture. Not yet a reviewed/fabrication-ready design.**  
**MCU:** CH32V006F8P7, QSOP24.  
**External interface retained:** 12 V input, three-digit 7-segment display, SET/+/- buttons, NTC sensor, relay contacts, optional buzzer, WCH-Link programming access.  
**Firmware:** direct W1209-PY behavior port, English only.

This note converts the connectivity draft into a concrete net-by-net proposal and a parts shortlist. Confirm manufacturer datasheets, exact package pinout, footprints, current/thermal ratings, and local safety requirements before release. In particular, verify the actual YSX-2381-6 display pinout/polarity against the physical part or its manufacturer drawing; do not assume a generic 2381-series pinout is exact.

## 1. Selected architecture

- Use **Murata OKI-78SR-5/1.5-W36-C** as the 12 V-to-5 V regulator module. Its wide input rating provides margin over a nominal 12 V adapter and avoids designing an unverified discrete switcher. Check the current module datasheet, input transients, and thermal derating; it is a module, not a substitute for input protection.
- Operate the CH32V006 at a regulated 5 V rail, subject to confirmation of the exact part's recommended VDD range and peripheral limits. Use separate local analog filtering/decoupling for VDDA.
- Assume a **common-cathode** display only if the installed YSX-2381-6 is verified as such. Use a high-side segment source driver and low-side digit switches. This is the conventional arrangement for common-cathode multiplexing.
- Use **TBD62783A** as the eight-channel high-side segment source driver (one channel per A–G/DP), with one series resistor per segment. Use three small NPN transistors as low-side digit-common switches.
- Use a logic-level N-MOSFET to switch the 12 V relay coil, with a flyback diode and gate pull-down. Fit a specified 12 V relay only after confirming coil current and contact rating for the target application.

## 2. Proposed MCU pin map

Package pin numbers follow the QSOP24 mapping provided for this design. Cross-check against WCH's latest CH32V006 datasheet before creating the footprint.

| MCU pin | MCU pad | Net | External connection |
|---:|---|---|---|
| 1 | PA1 | SEG_A_IN | TBD62783A IN1 |
| 2 | PA2 / ADC0 | NTC_SENSE | NTC divider ADC node |
| 3 | VSS | GND | Ground plane |
| 4 | PD4 / ADC7 | SEG_B_IN | TBD62783A IN2 |
| 5 | PD5 / ADC5 | SEG_C_IN | TBD62783A IN3 |
| 6 | PD6 / ADC6 | SEG_D_IN | TBD62783A IN4 |
| 7 | NRST | NRST | J4 programming header pin 4 |
| 8 | PD7 | SEG_E_IN | TBD62783A IN5 |
| 9 | PA4 | NC_PA4 | No connect in Rev A; test pad optional |
| 10 | VSSA | AGND | Quiet analog ground |
| 11 | VDDA | +5V_A | Filtered +5 V analog supply |
| 12 | PC0 | SEG_F_IN | TBD62783A IN6 |
| 13 | PC1 | SEG_G_IN | TBD62783A IN7 |
| 14 | PC2 | SEG_DP_IN | TBD62783A IN8 |
| 15 | PC3 | DIGIT_1_CTRL | QD1 base resistor |
| 16 | PC4 / ADC2 | DIGIT_2_CTRL | QD2 base resistor |
| 17 | PC5 | DIGIT_3_CTRL | QD3 base resistor |
| 18 | PC6 | BTN_SET_N | SET switch input |
| 19 | PC7 | BTN_PLUS_N | + switch input |
| 20 | VDD | +5V_LOGIC | Main 5 V supply |
| 21 | PD0 | BTN_MINUS_N | - switch input |
| 22 | PD1 / SWIO | SWIO | J4 programming header pin 3; reserve for debug |
| 23 | PD2 / ADC3 | RELAY_GATE | Q1 gate resistor |
| 24 | PD3 / ADC4 | BUZZER_CTRL | Optional Q2 base resistor |

## 3. Netlist-style connectivity

The following is a human-readable netlist for schematic entry. It uses logical component pin names; map them to the selected library symbols and footprints after checking each datasheet.

### Input protection and 5 V power

```text
J1.1  VIN_12V_RAW
J1.2  GND

VIN_12V_RAW -- F1.1
F1.2 -- VIN_FUSED
VIN_FUSED -- DREV.A
DREV.K -- +12V_PROT
DTVS.K -- +12V_PROT
DTVS.A -- GND
CIN1.1 -- +12V_PROT
CIN1.2 -- GND
CIN2.1 -- +12V_PROT
CIN2.2 -- GND

U1.VIN -- +12V_PROT
U1.GND -- GND
U1.VOUT -- +5V_LOGIC
COUT1.1 -- +5V_LOGIC
COUT1.2 -- GND
COUT2.1 -- +5V_LOGIC
COUT2.2 -- GND

+5V_LOGIC -- FB1.1
FB1.2 -- +5V_A
CVA1.1 -- +5V_A
CVA1.2 -- AGND
CVA2.1 -- +5V_A
CVA2.2 -- AGND
MCU.VDD (pin 20) -- +5V_LOGIC
MCU.VSS (pin 3) -- GND
MCU.VDDA (pin 11) -- +5V_A
MCU.VSSA (pin 10) -- AGND
AGND -- GND at a single quiet connection close to MCU/ADC return
```

U1 is the OKI-78SR-5/1.5-W36-C module. Its pin numbering is module-specific; use the Murata symbol/pinout, not guessed pin numbers. Follow its datasheet's required input/output capacitance and derating. CIN/COUT values below are recommended starting values and must meet the module datasheet requirements.

### NTC input

```text
RPU.1 -- +5V_A
RPU.2 -- NTC_SENSE
J2.1 -- NTC_SENSE
J2.2 -- AGND
C_NTC.1 -- NTC_SENSE
C_NTC.2 -- AGND
MCU.PA2/ADC0 (pin 2) -- NTC_SENSE
```

RPU is a replaceable 10.0 kOhm, 1% resistor by default. Populate C_NTC only after ADC settling/noise tests; use 10 nF as a footprint/value candidate. The sensor is connected from NTC_SENSE to AGND. P12 firmware configuration must match the physically fitted pull-up.

### Buttons

```text
+5V_LOGIC -- RSET 10k -- BTN_SET_N
BTN_SET_N -- SW_SET.1
SW_SET.2 -- GND
MCU.PC6 (pin 18) -- BTN_SET_N

+5V_LOGIC -- RPLUS 10k -- BTN_PLUS_N
BTN_PLUS_N -- SW_PLUS.1
SW_PLUS.2 -- GND
MCU.PC7 (pin 19) -- BTN_PLUS_N

+5V_LOGIC -- RMINUS 10k -- BTN_MINUS_N
BTN_MINUS_N -- SW_MINUS.1
SW_MINUS.2 -- GND
MCU.PD0 (pin 21) -- BTN_MINUS_N
```

Optional 4.7 nF debounce capacitors may be fitted from each input to GND; firmware debounce remains required. Do not populate large capacitors without checking button timing and input leakage.

### Display source driver and digit switches

Assumption: verified common-cathode 3-digit display. Display module pin mapping must be taken from its actual datasheet/continuity check and mapped to the nets below.

```text
MCU.PA1 (pin 1) -- U2.IN1
MCU.PD4 (pin 4) -- U2.IN2
MCU.PD5 (pin 5) -- U2.IN3
MCU.PD6 (pin 6) -- U2.IN4
MCU.PD7 (pin 8) -- U2.IN5
MCU.PC0 (pin 12) -- U2.IN6
MCU.PC1 (pin 13) -- U2.IN7
MCU.PC2 (pin 14) -- U2.IN8
U2.VCC -- +5V_LOGIC (only if the selected TBD62783A package/datasheet requires this supply pin)
U2.GND -- GND

U2.OUT1 -- RSEG_A 1.0k -- DISP.SEG_A
U2.OUT2 -- RSEG_B 1.0k -- DISP.SEG_B
U2.OUT3 -- RSEG_C 1.0k -- DISP.SEG_C
U2.OUT4 -- RSEG_D 1.0k -- DISP.SEG_D
U2.OUT5 -- RSEG_E 1.0k -- DISP.SEG_E
U2.OUT6 -- RSEG_F 1.0k -- DISP.SEG_F
U2.OUT7 -- RSEG_G 1.0k -- DISP.SEG_G
U2.OUT8 -- RSEG_DP 1.0k -- DISP.SEG_DP

MCU.PC3 (pin 15) -- RDIG1 2.2k -- QD1.B
MCU.PC4 (pin 16) -- RDIG2 2.2k -- QD2.B
MCU.PC5 (pin 17) -- RDIG3 2.2k -- QD3.B
QD1.E -- GND; QD2.E -- GND; QD3.E -- GND
QD1.C -- DISP.COM1; QD2.C -- DISP.COM2; QD3.C -- DISP.COM3
RPD1 100k: QD1.B -- GND
RPD2 100k: QD2.B -- GND
RPD3 100k: QD3.B -- GND
```

- U2 is TBD62783A (8-channel source driver), with package pinout, supply requirements, input thresholds, and output voltage drop checked against the exact Toshiba datasheet. Confirm that the chosen suffix/package exposes the assumed pins; connect any unused channels/pins per datasheet.
- QD1–QD3: MMBT2222A NPN in SOT-23 (or a qualified equivalent). Check package pinout for each manufacturer. Base resistors 2.2 kOhm are starting values; calculate base current against the maximum simultaneous LED segment load and MCU output limits.
- RSEG_A..DP: 1.0 kOhm, 1%, 0603, one per segment. This is a conservative brightness starting point, not a guaranteed final value. Calculate actual peak current using actual LED forward voltage, U2 output drop, supply tolerance, and multiplex duty; confirm brightness on the physical display. Never omit segment resistors.
- Ensure digit-select firmware blanks all digits while segment data changes. Hardware pull-downs keep all digit transistors off during reset.

If the installed display turns out common-anode, do not build this digit stage: the digit/segment driver polarities must be redesigned and the netlist revised.

### Relay driver and contact connector

```text
K1.COIL_A -- +12V_PROT
K1.COIL_B -- RELAY_COIL_LOW
D1.K -- +12V_PROT
D1.A -- RELAY_COIL_LOW
Q1.D -- RELAY_COIL_LOW
Q1.S -- GND
MCU.PD2 (pin 23) -- RG 100R -- Q1.G
RGS 100k: Q1.G -- GND

J3.1 -- K1.COM
J3.2 -- K1.NO
J3.3 -- K1.NC
```

- Q1: AO3400A N-channel MOSFET, SOT-23, candidate for a low-current 12 V relay coil; verify gate drive at the MCU's 5 V output, VDS/transient margin, SOA, thermal limits, pinout, and actual coil current. If relay current exceeds its design envelope, select another MOSFET from verified datasheets.
- D1: SS14 Schottky, cathode to +12V_PROT and anode to RELAY_COIL_LOW. Verify reverse-voltage/surge/current suitability for the selected coil.
- K1 candidate: Songle SRD-12VDC-SL-C 12 V coil, subject to confirming manufacturer authenticity, coil resistance/current, exact contact variant, physical footprint, and load rating. The existing W1209 T78 marking alone does not establish a safe contact rating.
- J3 contacts require application-specific creepage, clearance, fuse, terminal, and enclosure design. Do not connect mains until independently safety-reviewed.

### Optional buzzer

```text
MCU.PD3 (pin 24) -- RB 2.2k -- Q2.B
Q2.E -- GND
Q2.C -- BUZZER_NEG
BUZZER_POS -- +5V_LOGIC (only for a 5 V buzzer)
RBP 100k: Q2.B -- GND
```

Q2: MMBT2222A candidate. Select buzzer type/current explicitly. Add a diode across an inductive magnetic buzzer (cathode to positive); do not add one to a piezo element unless required by its datasheet. Passive piezo drive may need PWM capability; verify PD3 timer routing or use software toggling at an appropriate rate.

### WCH-Link programming header J4

| J4 pin | Silkscreen | Net |
|---:|---|---|
| 1 | VREF / +5V_TARGET | +5V_LOGIC reference only; do not power the board from WCH-Link unless its manual permits it |
| 2 | GND | GND |
| 3 | SWIO | PD1, MCU package pin 22 |
| 4 | NRST | NRST, MCU package pin 7 |

Keep PD1/SWIO free of loads and high-capacitance circuits. Check the exact WCH-Link model's required header pinout and target-voltage behavior before assembly. This is WCH-Link single-wire debug/programming, not ARM SWD.

## 4. Component list (proposed BOM)

| Ref | Qty | Part / value | Package / note |
|---|---:|---|---|
| U1 | 1 | Murata OKI-78SR-5/1.5-W36-C | SIP-3 switching regulator module, nominal 12 V in / 5 V out; check current datasheet and derating |
| F1 | 1 | 500 mA slow-blow or resettable fuse candidate | Set final rating from actual startup/relay/logic load and wire rating |
| DREV | 1 | SS34 | SMA Schottky reverse-polarity series protection candidate; check 12 V voltage drop/heat and surge |
| DTVS | 1 | SMBJ18A candidate | Across protected input; verify clamp vs adapter and U1 maximum input; omit/reselect after transient review |
| CIN1 | 1 | 10 uF, 35 V, X7R | 1210 or suitable footprint; module input bypass, verify effective capacitance under DC bias |
| CIN2 | 1 | 100 nF, 50 V, X7R | 0603; local high-frequency input bypass |
| COUT1 | 1 | 22 uF, 10 V, X7R | 0805/1206; check regulator module minimum capacitance and DC-bias derating |
| COUT2 | 1 | 100 nF, 10 V, X7R | 0603; local output bypass |
| FB1 | 1 | BLM21PG221SN1 (candidate) or equivalent | 0805 ferrite between +5V_LOGIC and +5V_A; verify current/DCR, can be 0-ohm/DNP if WCH advises direct tie |
| CVA1 | 1 | 1 uF, 10 V, X7R | 0603/0805, VDDA-to-AGND |
| CVA2 | 1 | 100 nF, 10 V, X7R | 0603, close to VDDA/VSSA |
| CVDD1 | 1 | 100 nF, 10 V, X7R | 0603, close to VDD/VSS |
| CVDD2 | 1 | 1 uF, 10 V, X7R | 0603/0805, close to MCU |
| U3 | 1 | CH32V006F8P7 | QSOP24; exact pinout/footprint verified to WCH datasheet |
| U2 | 1 | TBD62783A | 8-channel high-side source driver; exact package pinout/supply/input/output limits verified to Toshiba datasheet |
| DISP1 | 1 | YSX-2381-6 (existing part) | Verify actual common-cathode polarity, physical pin map, color/Vf, and current before schematic symbol/footprint release |
| RSEG_A..DP | 8 | 1.0 kOhm, 1% | 0603, one per segment; recalculate from actual display and driver drop |
| QD1..QD3 | 3 | MMBT2222A | SOT-23 NPN digit low-side switches; verify vendor-specific pinout |
| RDIG1..3 | 3 | 2.2 kOhm | 0603 base resistors; verify base current and GPIO current limits |
| RPD1..3 | 3 | 100 kOhm | 0603 digit-driver base pull-downs |
| RPU | 1 | 10.0 kOhm, 1% | 0603, removable NTC pull-up; default profile must agree |
| C_NTC | 1 | 10 nF, C0G/X7R | 0603 footprint, DNP until ADC/noise/settling validation |
| RSET/RPLUS/RMINUS | 3 | 10 kOhm, 1% | 0603 button pull-ups |
| SW_SET/SW_PLUS/SW_MINUS | 3 | Momentary switches | Match W1209 panel/board footprint |
| Q1 | 1 | AO3400A | SOT-23 N-MOSFET relay low-side switch; qualify for actual coil current and pinout |
| RG | 1 | 100 Ohm | 0603 Q1 gate series resistor |
| RGS | 1 | 100 kOhm | 0603 Q1 gate pull-down |
| D1 | 1 | SS14 | SMA Schottky relay coil flyback; recheck coil and transient requirements |
| K1 | 1 | SRD-12VDC-SL-C candidate | 12 V coil relay; confirm coil/contact variant, footprint, load and safety requirements |
| Q2 | 1 | MMBT2222A | SOT-23 optional buzzer NPN switch |
| RB | 1 | 2.2 kOhm | 0603 buzzer base resistor |
| RBP | 1 | 100 kOhm | 0603 buzzer base pull-down |
| J1 | 1 | 2-pin connector | 12 V input, current rating appropriate to relay/logic load |
| J2 | 1 | 2-pin sensor connector | NTC probe |
| J3 | 1 | 3-pin terminal block | COM/NO/NC; terminal and spacing application-rated |
| J4 | 1 | 1x4 2.54 mm header/test pads | VREF, GND, SWIO, NRST; check programmer cable orientation |

## 5. Regulator choice and constraints

U1 is provisionally specified as Murata OKI-78SR-5/1.5-W36-C because it accepts a wide DC input and supplies regulated 5 V at up to its rated current without a custom switcher compensation/inductor design. The relay coil is on +12V_PROT and is not included in the 5 V load. Before release:

1. Check the current Murata datasheet for input range, absolute maximum, output current derating, startup behavior, transient response, minimum/maximum capacitance, and recommended layout.
2. Confirm input transient suppression/clamp voltage cannot exceed the module's limits.
3. Calculate 5 V load worst case: MCU + display driver + three digits at chosen segment peak current/duty + optional buzzer + WCH-Link loading (normally do not power from header).
4. Check module thermal performance at the expected ambient and enclosure airflow. If input is a regulated 12 V adapter with known transients, document that assumption.
5. Set the input fuse and connectors based on measured input/startup current and wire rating.

## 6. Display driver selection and current setting

- U2 TBD62783A sources the eight segment anodes for a common-cathode display. Verify its exact datasheet logic supply and output current/voltage drop at the selected current. Its outputs are not substitutes for the eight segment resistors.
- QD1–QD3 MMBT2222A sink each common cathode. At a conservative 2–3 mA peak per segment, one digit's common current is at most about 24 mA for eight lit segments. Recalculate for actual firmware scan duty, brightness requirement, LED Vf, and resistor tolerance.
- With +5 V rail, red LED Vf around 1.8–2.2 V, a driver drop, and 1 kOhm series resistance, peak segment current is expected to be only a few mA. Measure actual brightness; do not assume generic 2381-series specifications apply to the installed YSX part.
- Confirm MCU output drive only sees U2 logic inputs and transistor bases, not display LED current. Confirm U2 inputs accept the CH32V006 5 V GPIO high level.
- The YSX-2381-6 exact pin numbering remains a must-check. In schematic, use nets SEG_A..DP and DIGIT_1..3 and fill the display symbol pin numbers only from an actual part drawing or continuity test.

## 7. Relay driver selection and safety

- AO3400A is a candidate N-MOSFET with gate connected to +5 V MCU logic through 100 Ohm. Verify on-resistance at the actual gate voltage, VDS margin including coil transients, temperature, and actual relay coil current.
- SS14 flyback diode is a candidate; verify repetitive coil energy and temperature. D1 polarity: cathode at +12V_PROT, anode at Q1 drain/RELAY_COIL_LOW.
- SRD-12VDC-SL-C is an example of a common 12 V relay, not a guaranteed equivalent to every T78-marked relay. Verify the exact relay datasheet and footprint. Assign contact ratings only from the actual relay's resistive/inductive load rating and board insulation design.
- Keep high-voltage contact routing physically separated from MCU/NTC/probe/programming circuitry. Add slots/keepouts/fusing and have the board reviewed for the intended voltage/current, pollution degree, material CTI, enclosure, and applicable safety standards. The present design note does not authorize mains use.

## 8. Programming header silkscreen

Use a keyed 1x4 footprint if practical; add a pin-1 triangle and silkscreen labels on both the PCB and schematic symbol:

```text
J4 (view from component side, keyed header orientation documented)
1  VREF   +5V_LOGIC target reference only
2  GND
3  SWIO   PD1
4  NRST
```

WCH-Link models/cables can differ: verify the exact tool header before manufacture. Do not power the board from VREF unless the tool's documentation explicitly permits it. Keep SWIO trace short and free of LED/buzzer/relay loads.

## 9. Pre-layout signoff checklist

- Confirm exact CH32V006F8P7 QSOP24 pin assignment, ADC0 on PA2, SWIO on PD1, supply limits, GPIO limits, and pin mux with WCH datasheet/reference manual.
- Confirm YSX-2381-6 common polarity, physical pinout, Vf and allowed current from the actual part/drawing; validate display symbol and footprint.
- Confirm U2 TBD62783A package pinout, supply, input thresholds, output ratings and output drop from Toshiba datasheet.
- Confirm U1 module limits/derating/capacitors, DREV/TVS compatibility, and worst-case 5 V load.
- Confirm Q1 gate performance and D1 coil energy with the selected relay.
- Run KiCad ERC/DRC; verify power-on defaults leave relay, digits, and buzzer off.
- Prototype at current-limited low voltage, test each display segment/digit and known NTC resistors, then exercise relay with unloaded contacts and a safe low-voltage dummy load.
- Do not connect mains until independent creepage/clearance and safety review is complete.

## References

- WCH CH32V006 pinout: https://chiptron.cz/pinouty/ch32v006f8p7-pinout/
- WCH CH32V006 docs/SDK: https://github.com/ch32-riscv-ug/CH32V006
- WCH official datasheet: https://www.wch-ic.com/downloads/CH32V006DS0_PDF.html
- WCH reference manual: https://www.wch.cn/downloads/CH32V00XRM_PDF.html
- Murata OKI-78SR-5/1.5-W36-C datasheet: https://www.murata.com/en-us/products/productdetail?partno=OKI-78SR-5%2F1.5-W36-C
- Toshiba TBD62783A datasheet/product page: https://toshiba.semicon-storage.com/us/semiconductor/product/linear-ics/transistor-arrays/detail.TBD62783AFG.html
- onsemi SS8050 reference (not used in primary BOM): https://www.onsemi.com/pdf/datasheet/ss8050-d.pdf
- YSX-2381-6 equivalent-display data; exact part must be verified: https://www.xlitx.com/datasheet/2381AS.pdf
- W1209-PY English behavior reference: https://github.com/Saco-de-patatas/W1209-PY/blob/main/README.EN.MD

**This proposal is a concrete starting BOM/netlist, not a released schematic or validated PCB. Part substitutions, final values, exact display pin numbering, mechanical fit, safety spacing, and prototype testing remain open engineering signoffs.**
