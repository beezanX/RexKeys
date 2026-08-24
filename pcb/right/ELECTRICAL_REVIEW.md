# Electrical Review — `key_right.kicad_sch`

**Date:** 2026-08-22
**Tool:** KiCad 10.0.3 (`kicad-cli` ERC + manual netlist analysis)
**Board:** Wireless split keyboard, right half (nRF52840 / IS31FL3733 / BQ24074 / USB-C)

## Verdict

**No electrical errors found.** The automated ERC passes with **0 errors**. All power, USB-C,
charger, MCU, I²C and key-matrix circuits were traced in the exported netlist and are wired
correctly. The 73 ERC *warnings* are cosmetic only (stale cached library symbols + one unconfigured
footprint library). A handful of design *considerations* are listed at the end — none are faults.

## What was checked

| Check | Result |
|---|---|
| KiCad ERC (all severities) | 0 errors, 73 warnings (cosmetic) |
| ERC with `single_global_label` re-enabled* | 0 errors — no orphaned/typo'd nets |
| Power-input pins driven (PWR_FLAG coverage) | OK — no undriven power pins |
| Unused pins | 38 No-Connect flags present; ERC clean |
| Manual netlist trace of all critical circuits | OK (details below) |

\* The project had `single_global_label`, `four_way_junction`, and `footprint_filter` set to
*ignore*. Because this design carries **100% of its connectivity through global labels**, a
single-instance label (i.e. a floating pin from a net-name typo) would be invisible. Re-running ERC
with that rule promoted to *error* still produced 0 violations — the net names are consistent.

## Circuit-by-circuit results

**System architecture:** USB-C `+5V` → BQ24074 (U4) power path → `VCC_SYS` → nRF52840 (U2)
internal high-voltage DC/DC (REG0) → `3V3`. LEDs driven from `VCC_SYS` via IS31FL3733 (U3).

- **USB-C (J3):** CC1 and CC2 each have their own **5.1 kΩ Rd to GND** (R2, R3) — correct
  sink/UFP configuration. D+/D− go to nRF pins 35/34 with both orientations tied (A6=B6, A7=B7),
  standard for a USB-2.0-only receptacle. VBUS → nRF pin 32 (5 V-tolerant detect) with 4.7 µF (C6). ✔
- **Charger (BQ24074, U4):** All program pins populated — ISET 590 Ω (≈1.51 A fast charge),
  ILIM 1.1 kΩ (≈1.46 A input limit), ITERM 3.0 kΩ, TMR 46.4 kΩ safety timer, TS 10 kΩ→GND.
  Input (C6 4.7 µF), OUT/`VCC_SYS` (1 µF+4.7 µF+100 nF+10 µF) and BAT (C9 4.7 µF) caps all present. ✔
- **nRF52840 (U2) power:** High-voltage DC/DC mode — VDDH on `VCC_SYS`, DCCH → **10 µH (L2)** →
  VDD (`3V3`). Decoupling: `VCC_SYS` 4.7 µF+100 nF, `3V3` 4.7 µF+1 µF+100 nF. Matches Nordic
  reference. ✔
- **I²C ×2:** LED bus (P0.26/P0.27 ↔ U3) and trackpad bus (P0.22/P0.23 ↔ J4) each have their own
  **4.7 kΩ pull-ups to 3V3** (R9/R10, R5/R6). ✔
- **Status/telemetry:** `BAT_CHG` (open-collector ~CHG) pulled to 3V3 via 10 kΩ; `PGOOD`
  (open-collector) via 100 kΩ; battery sense via 1 MΩ/1 MΩ divider to P0.02 (AIN0); status RGB
  (D50, common-anode to 3V3) sunk through 3×330 Ω to P1.11/12/13. ✔
- **Key matrix:** 6 columns (MCU GPIO → SW pin 1) × 4 rows, with a **per-key diode** (anode at
  switch, cathode to row) on every key — proper N-key-rollout diode matrix. 23 switches / 23 diodes,
  consistent. ✔
- **LED matrix (IS31FL3733):** RSET 20 kΩ, 5 SW rows (LED_R1–R5) × 15 CS columns used; unused
  SW/CS/SYNC/INTB pins No-Connect-flagged. ✔

## Design considerations (not errors)

1. **Charge current ≈1.51 A** (ISET = 590 Ω) sits at the BQ24074's 1.5 A ceiling. Confirm the QFN
   thermal design / charge de-rating at your worst-case (VIN−VBAT) drop.
2. **Input current limit ≈1.46 A** (ILIM = 1.1 kΩ) with **no CC-line current-advertisement
   detection** (CC pins carry only Rd; nothing reads the source's Rp). From a legacy/default USB-C
   source this exceeds the advertised contract. The BQ24074's DPPM prevents brown-out, but it is
   technically out of strict USB-C spec — fine for charging from a proper adapter.
3. **Battery temperature sensing disabled** — TS = 10 kΩ→GND is the datasheet "TS unused" wiring, so
   charging is never gated on battery temperature. Acceptable, but there is no over/under-temp
   charge protection.
4. **`3V3` is sourced entirely from the nRF52840 REG0 output.** Everything on 3V3 (status LED up to
   ~12 mA, trackpad, I²C pull-ups, IS31 digital core) draws from REG0 on top of the MCU's own
   consumption. Verify the external load fits REG0's headroom, particularly during radio TX.
5. **BAT_SENSE divider** has ~500 kΩ source impedance — configure a long SAADC acquisition time (and
   consider a small filter cap to `BAT_SENSE`) for a stable reading.
6. **Cosmetic ERC cleanup:** refresh the cached library symbols (VDD, R_Small, USB_C, LED_RGB show
   `lib_symbol_mismatch`) and add the `axseem` footprint library to project config to clear all 73
   warnings.
