# W1209 CH32V006F8P7 Rev A design specification

**Purpose:** one-page architecture, power budget, driver calculations, exact proposed MCU net map, and grouped BOM for schematic capture.  
**Status:** Rev A engineering proposal. It is not a production release: exact YSX-2381-6 display polarity/pinout, MCU electrical limits, relay variant, regulator derating, and enclosure/contact safety must be signed off from the actual parts and their datasheets.

## 1. One-page architecture

```text
J1: regulated 12 V adapter (recommended 12 V / 0.25 A minimum, final load check)
   -> F1 input fuse -> DREV reverse-polarity protection -> +12V_PROT
      |-> K1 12 V relay coil -> Q1 low-side N-MOSFET -> GND
      |        D1 flyback diode across coil
      |-> U1 Murata OKI-78SR-5/1.5-W36-C -> +5V_LOGIC
                |-> CH32V006F8P7 VDD (pin 20), display driver, buttons, optional buzzer
                |-> FB1 + local caps -> +5V_A -> VDDA (pin 11), NTC pull-up

3-digit common-cathode display (ASSUMPTION; verify actual display):
 MCU eight segment GPIOs -> U2 TBD62783A source outputs -> one R per segment -> display A..G,DP
 MCU three digit GPIOs -> base resistors -> QD1..QD3 NPN low-side switches -> display commons

NTC connector:
 +5V_A -> replaceable R_PULLUP (10k default) -> NTC_SENSE -> J2 NTC -> AGND
                                     |-> MCU PA2/ADC0 (pin 2)

Buttons: SET/+/- active-low inputs, 10k pull-up to +5V_LOGIC, switch to GND
Programming: J4 VREF(+5V reference), GND, SWIO(PD1 pin 22), NRST(pin 7)
```

Ground is a continuous ground plane with quiet analog routing; connect VSSA/AGND to GND as recommended by WCH, and route relay/display return currents away from the sensor return. The display and relay drivers default OFF during reset. Relay contacts are isolated from the low-voltage nets and need application-specific creepage, clearance, fuse, terminal, and enclosure design.

## 2. Power budget (design estimate, not measured)

Assumptions: 5.0 V regulated; 3-digit multiplex at 1/3 duty; 1.0 kOhm segment resistors; 8 segments lit for the active digit; MCU power allowance is deliberately conservative; optional 5 V buzzer load limited to 30 mA. The actual MCUs, display and buzzer datasheets take precedence.

| Load | Estimated worst case | Basis / note |
|---|---:|---|
| CH32V006 + clock/peripherals | 30 mA at 5 V | Board budget allowance; replace with measured/datasheet worst-case current |
| Display segments, instantaneous | 8 x 3 mA = 24 mA | Conservative 1k series resistors; only one digit enabled at a time |
| Display segments, average | about 8 mA | 24 mA x one-third duty; excludes driver quiescent/leakage |
| TBD62783A + digit transistor control | 5 mA allowance | Confirm with exact driver input/leakage characteristics |
| Three button pull-ups | 1.5 mA max | 3 x 5 V / 10k with all buttons held |
| NTC divider | <=0.5 mA | 5 V / 10k when sensor is shorted; at 10k NTC current is 0.25 mA |
| Optional buzzer | 30 mA | Choose a 5 V part with <=30 mA rated current or recalculate Q2 and regulator |
| **Total +5 V estimated** | **about 75 mA** | Includes buzzer; use >=150 mA design allowance pending measurement |
| 12 V relay coil | about 38 mA | If chosen coil is verified as 12 V, 320 Ohm; I=12/320. Confirm actual relay |
| **12 V adapter recommendation** | **12 V, >=0.25 A regulated** | Margin over logic input power + coil; final value must include regulator efficiency, startup, adapter tolerance and ambient |

At estimated 75 mA output, 5 V load power is 0.375 W. At 12 V and an assumed 80% conversion efficiency, the logic input is about 39 mA. Adding a 38 mA relay coil gives about 77 mA steady-state input, before startup/transients and margin. A 0.25 A adapter provides useful margin, but confirm with prototype measurements. Relay contact load is a separate circuit and does not count as MCU/adapter coil current unless load is sourced by the board.

### Display-current check

Use one resistor per segment. Preliminary calculation for a red LED:

`Iseg_peak = (V5_max - Vf_min - Vdriver_drop - Vdigit_switch_drop) / Rseg`

Using conservative placeholders `V5_max=5.25 V`, `Vf_min=1.6 V`, `Vdriver_drop=0.5 V`, `Vdigit_switch_drop=0.2 V`, and `Rseg=1.0 kOhm` gives approximately `2.95 mA` peak. The exact Vf and driver drop must be taken from actual datasheets. At 1/3 duty, one continuously lit segment averages about 1 mA. Increase brightness only after confirming the display rating, TBD62783A output rating/drop, NPN dissipation, multiplex duty, and MCU input drive compatibility. Do not rely on generic 2381AS figures as an exact YSX-2381-6 specification.

## 3. Driver-stage calculations and selections

### Display: common-cathode assumption

- **Segment source:** U2 TBD62783A, 8 channels; use one channel per A–G/DP. Each output connects through an individual 1.0 kOhm, 1% resistor to the corresponding display segment anode. Confirm the selected TBD62783A suffix's logic supply/pinout, input thresholds, output-current rating, and voltage drop.
- **Digit sinks:** QD1–QD3 MMBT2222A NPN low-side switches, one per digit common cathode. `MCU GPIO -> 2.2k -> base`; 100k base-to-GND keeps each digit off at reset. Approximate base current with a 5 V high and 0.8 V base: `(5-0.8)/2.2k = 1.9mA`. At ~24mA worst-case digit current, forced beta is about 13; verify transistor saturation with its datasheet and maintain GPIO current margin.
- Display firmware blanks all digits before segment changes. If physical display is common-anode, stop and redesign display stage polarity; do not populate this proposed circuit.
- Exact YSX-2381-6 footprint/pin-to-segment assignment remains **unresolved** until checked against actual display manufacturer drawing or bench continuity/LED test.

### Relay

- **Q1:** AO3400A candidate N-channel low-side switch, source to GND, drain to relay coil low-side, gate from PD2 through 100 Ohm, 100k gate-to-source pull-down. Verify AO3400A RDS(on) at 4.5–5 V gate drive, VDS/transient rating, pinout, and thermal performance.
- **D1:** SS14 candidate flyback diode directly across coil, cathode to +12V_PROT, anode to coil-low/Q1-drain. Verify repetitive coil current/energy and release-time needs.
- **K1:** use the exact relay model only after confirming coil voltage/current/resistance, package pinout, contact form, contact/load ratings and insulation spacing. A T78 marking does not establish a safe mains rating.
- Example coil calculation only: 12 V / 320 Ohm = 37.5 mA. Measure/verify the selected relay; do not assume all W1209 relays match.

### Optional buzzer

Q2 MMBT2222A NPN low-side driver candidate: PD3 through 2.2k to base, 100k base pull-down, emitter GND, collector to buzzer negative, buzzer positive to its rated supply. With a 5 V buzzer <=30 mA, estimated base current is 1.9 mA and forced beta <=16. Verify Q2 saturation and pinout. Add a flyback diode only for an inductive/magnetic buzzer, cathode at positive supply; passive piezo may require PWM.

## 4. Exact proposed CH32V006F8P7 QSOP24 allocation

Pin numbering/signal names below are the project's current intended map; verify against the exact WCH package pin assignment before symbol/footprint release.

| QSOP24 pin | MCU pin | Net | Connection |
|---:|---|---|---|
| 1 | PA1 | SEG_A_IN | U2 segment input 1 |
| 2 | PA2 / ADC0 | NTC_SENSE | NTC divider and ADC |
| 3 | VSS | GND | Ground |
| 4 | PD4 / ADC7 | SEG_B_IN | U2 segment input 2 |
| 5 | PD5 / ADC5 | SEG_C_IN | U2 segment input 3 |
| 6 | PD6 / ADC6 | SEG_D_IN | U2 segment input 4 |
| 7 | NRST | NRST | J4 pin 4 |
| 8 | PD7 | SEG_E_IN | U2 segment input 5 |
| 9 | PA4 | NC_PA4 | No connect in this revision |
| 10 | VSSA | AGND | Quiet analog return, joined per WCH guidance |
| 11 | VDDA | +5V_A | Filtered/decoupled analog rail |
| 12 | PC0 | SEG_F_IN | U2 segment input 6 |
| 13 | PC1 | SEG_G_IN | U2 segment input 7 |
| 14 | PC2 | SEG_DP_IN | U2 segment input 8 |
| 15 | PC3 | DIGIT_1_CTRL | QD1 base resistor |
| 16 | PC4 / ADC2 | DIGIT_2_CTRL | QD2 base resistor; ADC function unused |
| 17 | PC5 | DIGIT_3_CTRL | QD3 base resistor |
| 18 | PC6 | BTN_SET_N | SET active-low button |
| 19 | PC7 | BTN_PLUS_N | + active-low button |
| 20 | VDD | +5V_LOGIC | Main MCU supply |
| 21 | PD0 | BTN_MINUS_N | - active-low button |
| 22 | PD1 / SWIO | SWIO | J4 pin 3; reserve for WCH-Link |
| 23 | PD2 / ADC3 | RELAY_GATE | Q1 gate resistor |
| 24 | PD3 / ADC4 | BUZZER_CTRL | Optional Q2 base resistor |

## 5. Grouped Rev A BOM

These are selected candidates, not a fully qualified final purchasing BOM. Check orderable suffixes, footprints, lifecycle, manufacturer datasheets and local availability before release.

### Power

| Ref | Qty | Candidate | Notes |
|---|---:|---|---|
| J1 | 1 | 2-pin connector, >=0.25 A | 12 V regulated adapter input; select pitch/current/locking to enclosure |
| F1 | 1 | 250 mA time-delay fuse candidate | Re-rate from measured startup current, relay coil and wiring; do not treat as contact-load fuse |
| DREV | 1 | SS34 | Series reverse-polarity protection candidate; check drop/heat at worst input current |
| DTVS | 1 | SMBJ18A candidate, conditional | Select only after adapter maximum and regulator transient limits are reconciled; otherwise leave DNP pending review |
| U1 | 1 | Murata OKI-78SR-5/1.5-W36-C | 7–36 V input, fixed 5 V, 1.5 A nominal class; verify current datasheet derating/pinout and regulator module thermal conditions |
| CIN1 | 1 | 10 uF, >=35 V, X7R | Input local bulk; effective capacitance under DC bias matters |
| CIN2 | 1 | 100 nF, >=50 V, X7R | Input high-frequency bypass |
| COUT1 | 1 | 22 uF, >=10 V, X7R | Output bulk; verify module recommendations and effective capacitance |
| COUT2 | 1 | 100 nF, >=10 V, X7R | Output bypass |
| FB1 | 1 | BLM21PG221SN1 or 0 Ohm link | +5V_LOGIC to +5V_A only if current/DCR and WCH analog guidance allow; include bypass option |
| CVDD1/CVA2 | 2 | 100 nF, X7R | Local VDD/VSS and VDDA/VSSA bypass capacitors |
| CVDD2/CVA1 | 2 | 1 uF, X7R | Local MCU/analog bulk bypass |

### Logic and NTC

| Ref | Qty | Candidate | Notes |
|---|---:|---|---|
| U3 | 1 | CH32V006F8P7 | QSOP24, verify exact package/footprint and WCH pin table |
| RPU | 1 | 10.0 kOhm, 1% | Replaceable NTC pull-up; default firmware R_PULLUP must match |
| C_NTC | 1 | 10 nF, DNP initially | Optional ADC input filter; confirm sample settling and measured response |
| RSET/RPLUS/RMINUS | 3 | 10 kOhm, 1% | External button pull-ups; GPIO internal pulls set consistently |
| SW_SET/SW_PLUS/SW_MINUS | 3 | Momentary switches | Match W1209 mechanical layout |

### Display

| Ref | Qty | Candidate | Notes |
|---|---:|---|---|
| DISP1 | 1 | Existing YSX-2381-6 | Confirm physical pinout, polarity, Vf/current with actual part before symbol/footprint finalization |
| U2 | 1 | Toshiba TBD62783A (exact suffix to select) | 8-channel source driver for common-cathode segment anodes; verify datasheet pinout and ratings |
| RSEG_A..DP | 8 | 1.0 kOhm, 1%, 0603 | One resistor per segment; initial low-current value, recalc/bench-tune for brightness |
| QD1..QD3 | 3 | MMBT2222A | Low-side digit common switches; verify vendor package pinout |
| RDIG1..3 | 3 | 2.2 kOhm | NPN base resistors; ~1.9 mA base-current estimate at 5 V |
| RPD1..3 | 3 | 100 kOhm | Digit base pull-downs, default all digits off |

### Relay and buzzer

| Ref | Qty | Candidate | Notes |
|---|---:|---|---|
| K1 | 1 | Verified 12 V coil relay (SRD-12VDC-SL-C example only) | Actual contact rating, coil current/resistance, pinout and insulation must be established |
| Q1 | 1 | AO3400A | Relay low-side N-MOSFET candidate; verify RDS(on) at 5 V, VDS/transient rating and package pinout |
| RG/RGS | 1 each | 100 Ohm / 100 kOhm | Gate resistor / default-off gate pull-down |
| D1 | 1 | SS14 | Relay coil flyback candidate, cathode to +12V_PROT |
| Q2 | 1 | MMBT2222A, optional | Buzzer low-side transistor for chosen <=30 mA buzzer |
| RB/RBP | 1 each | 2.2 kOhm / 100 kOhm | Buzzer base resistor / default-off pull-down |
| BZ1 | 1 | 5 V buzzer, <=30 mA, optional | Choose active/passive type and firmware drive method; verify current |

### Connectors and programming

| Ref | Qty | Candidate | Notes |
|---|---:|---|---|
| J2 | 1 | 2-pin sensor connector | NTC_SENSE / AGND; maintain low-voltage-only wiring |
| J3 | 1 | 3-pole relay terminal | COM/NO/NC; voltage/current/clearance are application-specific and require safety review |
| J4 | 1 | Keyed 1x4 2.54 mm header or test pads | Pin 1 VREF(+5V reference), 2 GND, 3 SWIO/PD1, 4 NRST |

## 6. Programming-header labeling

Silkscreen J4 with pin-1 triangle and these labels, oriented to the documented WCH-Link cable view:

```text
1 VREF   +5V target reference; not a power output unless programmer permits
2 GND
3 SWIO   PD1
4 NRST
```

Verify the exact WCH-Link model's cable/header orientation and target-reference requirements. Keep SWIO unloaded and the trace short. This is WCH-Link single-wire programming/debug, not ARM SWD.

## 7. Release blockers

1. Confirm CH32V006F8P7 QSOP24 pin numbering, supply range, ADC input limits, GPIO current limits, SWIO behavior and alternate functions in WCH's official datasheet/reference manual. Do not rely on a TSSOP20 drawing for a QSOP24 footprint.
2. Confirm YSX-2381-6 physical pinout, common polarity, LED Vf and current from the actual component drawing/measurement. Revise display driver if it is not common-cathode.
3. Calculate final resistor current and brightness from the actual display Vf range, driver drop and 5 V tolerance; test all 8 segments at multiplex duty.
4. Check Murata U1 input maximum/transient behavior and thermal/current derating; calculate TVS clamp against actual adapter and U1 rating. Finalize F1 to adapter, inrush and wiring.
5. Verify relay coil and contact details from the chosen relay's actual datasheet; perform separate mains-contact safety review before use.
6. Build a prototype and measure current, rail ripple, NTC ADC noise, relay switching transients, display brightness and reset-safe states. Then run ERC/DRC on the actual KiCad design.

## References

- CH32V006 pinout reference: https://chiptron.cz/pinouty/ch32v006f8p7-pinout/
- WCH CH32V006 docs/SDK: https://github.com/ch32-riscv-ug/CH32V006
- WCH datasheet: https://www.wch-ic.com/downloads/CH32V006DS0_PDF.html
- WCH reference manual: https://www.wch.cn/downloads/CH32V00XRM_PDF.html
- Murata OKI-78SR series datasheet: https://www.murata.com/en-us/products/productdetail?partno=OKI-78SR-5%2F1.5-W36-C
- Toshiba TBD62783A product/datasheet: https://toshiba.semicon-storage.com/us/semiconductor/product/linear-ics/transistor-arrays/detail.TBD62783AFG.html
- YSX-2381-6 generic-equivalent data only (not confirmation of exact device): https://www.xlitx.com/datasheet/2381AS.pdf
- W1209-PY English behavior: https://github.com/Saco-de-patatas/W1209-PY/blob/main/README.EN.MD

**This is a schematic-capture design proposal with a bounded estimated power budget, not a validated final circuit, released purchasing BOM, fabrication-ready PCB, or safety certification.**
