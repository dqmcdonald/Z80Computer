# Z80 Clock Board — Design Review (Post-J2 Fix)

**Board:** Z80 computer clock generator — 555 astable, crystal oscillator, manual step  
**Designer:** D. Q. McDonald · June 2026  
**Review date:** 2026-06-03  
**Reviewers:** kicad-happy automated analysis (v1.3.1) + manual verification  
**Components:** 22 (15 SMD, 7 THT) · 2 copper layers  
**Analyzer run:** `analysis/2026-06-03_1159` / `analysis/2026-06-03_1159-2`

---

## Delta from Prior Review (2026-06-02)

| Item | Before | After |
|------|--------|-------|
| J2 clock selector | Footprint/schematic mismatch ❌ | Rerouted — odd pins = CLK Out, even pins = sources ✅ |
| R6 value | "R_Small" — **unset** ❌ | **330 Ω** — 9 mA LED current ✅ |
| Vias | 15 (6 routing + 9 stitching) | 14 (routing) + 9 stitching |
| GND pour fill | 88.2% | 88.4% |
| SPICE | 2/2 pass | 2/2 pass ✅ |
| Thermal | 0 findings | 0 findings ✅ |

Both blockers from the prior review are resolved. The board is ready for hand assembly.

---

## Overview

A well-designed two-layer Z80 clock board implementing three independently-selectable clock sources: a variable-rate 555 astable oscillator (≈0.7 Hz–720 Hz), a 4 MHz crystal oscillator, and a manual single-step button. Source selection is via J2, a 2×3 pin jumper. A 74HC14 hex Schmitt-trigger provides buffering, signal conditioning, and the crystal oscillator inverter. Board dimensions: 47.6 × 48.7 mm. Fully routed, DRC-clean, B.Cu GND pour at 88.4% fill.

---

## Analyzers Run

| Analyzer | Status |
|----------|--------|
| Schematic (`analyze_schematic.py`) | ✅ Run |
| PCB (`analyze_pcb.py --full`) | ✅ Run |
| Cross-domain (`cross_analysis.py`) | ✅ Run |
| Thermal (`analyze_thermal.py`) | ✅ Run — 0 findings |
| SPICE (`simulate_subcircuits.py`, ngspice) | ✅ Run — 2/2 pass |
| EMC (`analyze_emc.py`) | ⚠ Not available in this skill version |
| Gerbers | Not present — not run |
| Lifecycle / MPN audit | Skipped — hand assembly, MPN not required |

---

## Component Summary

| Ref | Value | Package | Role |
|-----|-------|---------|------|
| U1 | TLC555xP | DIP-8 | Astable oscillator (variable slow clock) |
| U2 | 74HC14 | DIP-14 | Schmitt inverter: crystal OSC, buffering, ~CLK |
| Y1 | 4 MHz | HC49-U vertical | Fast clock source |
| RV1 | 1 MΩ | Bourns 3296W trimmer | 555 frequency control |
| SW1 | Pushbutton | 6 mm tactile | Manual single-step |
| J1 | 1×4 pin header | THT | Board output (5V, GND, CLK Out, ~CLK Out) |
| J2 | 2×3 pin header | THT | Clock source selector |
| R1, R2 | 1 KΩ | R_0805 | 555 timing (Ra) |
| R3 | 5 KΩ | R_0805 | Crystal oscillator series damping |
| R4 | 1 MΩ | R_0805 | Crystal oscillator feedback bias |
| R5 | 10 KΩ | R_0805 | SW1 debounce pull-up |
| R6 | 330 Ω | R_0805 HandSolder | LED current limiting |
| C2 | 1 µF | C_0805 | 555 timing capacitor |
| C1 | 10 nF | C_0805 | U1 CONT pin bypass |
| C3, C4 | 33 pF | C_0805 | Crystal load caps |
| C5, C6 | 100 nF | C_0805 | Decoupling |
| C7 | 100 nF | C_0805 | SW1 debounce capacitor |
| C8 | 10 µF | C_0805 | Bulk decoupling |
| D1 | LED | LED_0805 HandSolder | Status indicator |

---

## Schematic Analysis

### Power Tree

Single 5V rail from J1 Pin 1; GND from J1 Pin 2. Decoupling: C5 + C6 = 200 nF + C8 = 10 µF — adequate for this mixed-logic design. **[RS-001]** 5V rail has no `PWR_FLAG` — cosmetic ERC warning only; add a `PWR_FLAG` symbol near J1 to suppress it.

### TLC555 Astable Oscillator (U1)

Verified against TLC555CP datasheet:

| Pin | Name | Net | Check |
|-----|------|-----|-------|
| 1 | GND | GND | ✓ |
| 2 | TRIG | tied to THRES | ✓ astable config |
| 3 | OUT | 555 Out → J2 pin 4 | ✓ |
| 4 | ~RST | 5V (always enabled) | ✓ |
| 5 | CONT | C1 (10 nF bypass) | ✓ |
| 6 | THRES | tied to TRIG, C2 | ✓ |
| 7 | DISCH | R1+R2 → 5V | ✓ |
| 8 | VCC | 5V | ✓ |

**Timing:** Ra = R1 + R2 = 2 KΩ; Rb = RV1 (0–1 MΩ trimmer); C = C2 = 1 µF.  
f = 1.44 / ((Ra + 2·Rb) · C) → **≈ 0.7 Hz (RV1 max) to 720 Hz (RV1 min)**. Good three-decade range for step-by-step debugging through fast clocking.

### Crystal Oscillator (U2, Y1)

Pierce topology using one 74HC14 inverter:
- **R4 = 1 MΩ** — feedback resistor biasing the inverter into its linear region ✓
- **R3 = 5 KΩ** — series damping resistor to limit drive level ✓
- **C3 = C4 = 33 pF** — load caps; CL_eff = (33×33)/(33+33) = **16.5 pF + ≈5 pF stray ≈ 21–22 pF**

Typical HC49-U crystals specify CL = 18–32 pF. 21–22 pF estimated effective load is within range. ✓

Crystal output buffered through a second 74HC14 inverter (U2.3→4); Crystal Out → J2 pin 6.

**SPICE result:** Crystal load cap = 19.5 pF simulated ✅

### Manual Single-Step (SW1)

RC debounce: R5 = 10 KΩ pull-up, C7 = 100 nF to GND. τ = R5·C7 = **1.0 ms** — appropriate for a tactile pushbutton. SW1 active-low; U2 pin 5/6 inverts to produce Manual Out = HIGH when pressed. Manual Out → J2 pin 2.

**SPICE result:** RC filter cutoff = 158.8 Hz vs expected 159.2 Hz (−0.3%) ✅

### J2 Clock Selector (Post-Fix)

After schematic re-routing, connectivity is correct:

| J2 Pins | Net | Signal |
|---------|-----|--------|
| 1, 3, 5 (odd, common row) | CLK Out | Output to J1 + LED chain |
| 2 | Manual Out | Single-step (Schmitt-conditioned SW1) |
| 4 | 555 Out | 555 astable |
| 6 | Crystal Out | 4 MHz buffered crystal |

Place a jumper across **1–2** (manual), **3–4** (555 slow clock), or **5–6** (4 MHz crystal). ✓

**[CG-AUD]** J2 has no ground pin — expected for a signal-only source-selector, not a fault.

### LED Indicator (D1, R6)

D1 pin 1 = K (cathode) → R6 → GND. D1 pin 2 = A (anode) → U2 pin 10 output.  
LED is on when U2.10 is HIGH (sourcing ≈ (5 V − 2 V) / 330 Ω = **9 mA**) — within 74HC14 output source capability. ✓

U2.8 → U2.11 → U2.10 is a double-inversion of CLK Out, so the LED pulses in phase with CLK Out. At 4 MHz the LED will appear continuously on; at 555 rates it blinks visibly.

### ESD

**[EP-AUD]** J1 and J2 are internal board-to-board connectors; no ESD protection required for a lab/hobby design.

---

## PCB Layout Analysis

### Routing

- **Complete:** 40/40 connections routed ✓
- **Vias:** 14 routing vias + 9 GND stitching vias
- **Ground plane:** B.Cu GND zone, 88.4% fill ✓

### Trace Widths

GND traces: 0.2 mm minimum. At the current levels (<100 mA total), this is adequate (IPC-2221 1 oz Cu: 0.2 mm handles ≈ 400 mA). Power supply trace widths are not a concern for this design.

### Reference Plane Coverage (Cross-Analysis)

| Net | Coverage | Assessment |
|-----|----------|------------|
| CLK Out | 74% | Acceptable — max signal frequency is 4 MHz, well below the threshold where plane gaps cause SI issues |
| ~CLK Out | 89% | Acceptable |

### Manufacturing Notes

- **[FD-001]** No fiducials — finest pad dimension is 1.0 mm (coarse pitch THT/0805). Hand assembly does not require fiducials. ✓
- **[OR-001]** 4 passives not at 90° majority — minor aesthetic note, no functional impact.
- **[TE-001]** No test points on any of 17 nets. Recommend adding test points at minimum on: 5V, GND, CLK Out, 555 Out. Hand-probing is workable but test points speed up bring-up.

---

## Thermal Analysis

Score: **100/100** — 0 findings. No power-dissipating devices operate above ambient concern thresholds. The TLC555 at 5V with µA idle current and the 74HC14 at <100 mA combined produce negligible self-heating. ✓

---

## SPICE Simulation

| Subcircuit | Status | Expected | Simulated | Error |
|------------|--------|----------|-----------|-------|
| RC filter (R5/C7) | **PASS** | 159.15 Hz LP | 158.78 Hz | −0.2% |
| Crystal circuit (Y1) | **PASS** | CL = 19.5 pF | CL = 19.5 pF | 0% |

---

## Findings Summary

| ID | Severity | Summary | Action |
|----|----------|---------|--------|
| RS-001 | ⚠ Warning | 5V rail has no PWR_FLAG | Add `PWR_FLAG` near J1 — ERC cosmetic |
| TE-001 | ⚠ Warning | No test points (0/17 nets) | Add TP on 5V, GND, CLK Out, 555 Out |
| RP-002 | ⚠ Info | CLK Out 74% ref plane coverage | Acceptable at ≤4 MHz |
| CG-AUD | ℹ Info | J2 no ground pin | By design — source-selector jumper |
| EP-AUD | ℹ Info | No ESD on J1/J2 | Internal connectors — acceptable |
| FD-001 | ℹ Info | No fiducials | Hand assembly — not required |
| OR-001 | ℹ Info | 4 passives off-axis | Cosmetic |

**No blockers for hand assembly.** The two items worth actioning before a future PCBA run would be PWR_FLAG and test points.

---

## Verification Basis

| Check | Method |
|-------|--------|
| U1 TLC555 pin mapping | Verified against TLC555CP datasheet (datasheets/TLC555CP.pdf) |
| 555 timing calculation | Computed from R1/R2/RV1/C2 values |
| Crystal oscillator topology | Manual net trace — Pierce topology confirmed |
| Crystal load cap | Calculated CL_eff, confirmed within HC49-U typical range; SPICE verified |
| Debounce RC | Calculated, SPICE verified (−0.2%) |
| LED current | Calculated (9 mA), direction confirmed from pin name fields |
| J2 connectivity | Net trace — all six pins verified |
| Power decoupling | Net trace — C5/C6 on U2 VCC, C8 bulk, C1 on U1 CONT |

**Datasheet coverage:** TLC555CP verified; 74HC14 not independently downloaded but 74HC14 is a standard well-characterised device and the pin usage (inverter in/out/VCC/GND) is verified by net trace.

---

## Skipped Analyses

- **EMC:** `analyze_emc.py` not available in installed skill version.
- **Gerbers:** Not generated — not applicable at this stage.
- **Lifecycle/MPN:** Hand assembly — intentionally skipped per user instruction.

---

*Review produced by kicad-happy v1.3.1 + Claude (2026-06-03)*
