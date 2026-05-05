# Power tree subsystem

## Role

Convert 5V_BOARD (output of the input OR) into all the rails the i.MX 6ULL and peripherals consume.

## Architecture

```
5V_BOARD ──┬── TPS563201 buck → 3V3      (1A,  most NVCC_*, ETH PHY, audio, CH340N)
           ├── TPS563201 buck → 1V4      (600 mA, VDDARM_IN + VDDSOC_IN)
           ├── TPS563201 buck → 1V35     (400 mA, NVCC_DRAM for DDR3L)
           ├── LDK220 LDO    → 3V SNVS   (1 mA, VDD_SNVS_IN, with CR2032 backup)
           │
3V3 ──── TLV75901 LDO → 1V8              (300 mA, NVCC_SD card option, NVCC_NAND)
1V35 ──── TPS51200 → 0.675V VTT          (sink/source, DDR3L termination)
```

Three TPS563201 instances → one LCSC line item (commonization).

## i.MX 6ULL rail map

From `imx6ull-MCIMX6Y2CVM08AB-datasheet.pdf` §3.1 Power Supplies and §4.1 Chip-Level Conditions:

| Rail | i.MX pins | V | I (typ) | Source |
|---|---|---|---|---|
| VDD_SNVS_IN | SNVS supply | 2.8–3.3V | <1 mA | LDK220 + coin cell |
| VDD_HIGH_IN | high-voltage analog | 2.8–3.3V | low | 3V3 buck |
| VDDARM_IN | ARM core | 1.275–1.5V | up to 0.5A | 1V4 buck |
| VDDSOC_IN | SoC + GPU | 1.275–1.5V | up to 0.4A | 1V4 buck (same as ARM, allowed per datasheet) |
| NVCC_DRAM | DDR3L IO | 1.28–1.45V | up to 0.4A | 1V35 buck |
| NVCC_SD1 | SDcard IO | 1.65–1.95V or 2.7–3.6V | <100 mA | 1V8 LDO (or 3V3 — switch via PMIC; here just 1V8 for simplicity) |
| NVCC_ENET1 | Ethernet IO | 1.65–1.95V or 2.7–3.6V | <50 mA | 3V3 |
| NVCC_NAND, NVCC_JTAG, etc. | various | 3V3 | <50 mA each | 3V3 |
| USB_VBUS | USB OTG charge pump in | 4.0–5.5V | <10 mA | 5V_BOARD |

## First-principles design notes

### Why three instances of the same buck (TPS563201)

**Commonization.** TPS563201 spans 0.76–7V output, up to 3A — covers all three sub-5V rails (3V3, 1V4, 1V35) with one part in the BOM. Different feedback dividers per instance set the output voltage; everything else (compensation, inductor 2.2 µH, switching freq 580 kHz) is identical. Reduces JLC setup fee by 2× $3 vs three different bucks.

### Buck duty-cycle sanity check (first principles)

| Rail | Vin | Vout | Duty | Ton @ 580 kHz | TPS563201 Tmin (~110 ns) |
|---|---|---|---|---|---|
| 3V3 | 5V | 3.3V | 0.66 | 1138 ns | OK |
| 1V4 | 5V | 1.4V | 0.28 | 482 ns | OK |
| 1V35 | 5V | 1.35V | 0.27 | 466 ns | OK |

All three duty cycles produce Ton well above the controller minimum on-time. If we ever scale Vin up to 12V the 1V35 duty becomes 0.11 → Ton 194 ns, still OK but margins thin. For this design (5V input) we're comfortable.

### Why 1V8 from 3V3 LDO instead of dedicated buck

300 mA peak at 1.5V drop = **0.45 W** dissipation in the LDO. SOT-23-5 thermal limit is ~0.5 W. Marginal. Will derate — assume typical SD/eMMC traffic is 100 mA average, peak briefly at 300 mA. Could add a heat-spreading pour. A dedicated buck would save the dissipation but adds another inductor + 2× R + bulk caps + EN logic = more BOM, more area, more failure surface for a 0.45W problem.

**TODO:** confirm TLV75901 thermal pad is tied to GND copper pour; add 4–6 thermal vias to bottom plane.

### VTT termination — must be sink/source

DDR3L has bidirectional bus. VTT termination network (typ 39Ω to VTT on each DQ line) sinks current when the line is low and sources when high. A simple LDO can only source — it'll regulate when the bus is mostly high but lose voltage when the bus is mostly low. Need a sink/source like **TPS51200** (or SY56012 fallback). TPS51200 needs `ato create part` (not in registry).

VTT must be **VDDQ/2 = 0.675V** with VTTREF (separate output) supplying ~0.5% accurate reference for DDR3 receivers.

### VDD_SNVS — coin cell switchover

Always-on rail. Powers RTC + 4 KB secure RAM. Rule: keep board's VDD_SNVS at 3.0–3.3V when external power is present; switch to coin cell (2.5–3.3V) when bench/USB-C disconnected. **LDK220 + Schottky-OR with coin cell** is the standard. **TODO:** add the coin cell + Schottky in Debug subsystem.

### Power-on sequencing — known limitation

i.MX 6ULL datasheet §3.1 specifies: VDD_SNVS_IN → VDD_HIGH_IN → VDDARM/VDDSOC → NVCC_DRAM → POR_B release. Strict ordering helps avoid latch-up.

For prototype simplicity, all three TPS563201 EN pins are tied high via the package's default 10 kΩ pull-up to Vin → all bucks soft-start simultaneously. The 580 kHz buck soft-start (~1 ms) is much faster than i.MX's POR debounce (handled by the **TPS3808** or similar reset supervisor we'll add to Debug). The reset supervisor holds POR_B asserted until 3V3 is good — by which time all other rails have settled.

If first-article testing shows PoR-related failures, we'll revisit with proper PG-daisy-chain or RC delays on EN pins. **TODO: add TPS3808 to Debug subsystem.**

## Module API

```ato
module PowerTree:
    power_5v   : ElectricPower      # in
    power_3v3  : ElectricPower      # out
    power_1v4  : ElectricPower      # VDDARM/VDDSOC
    power_1v35 : ElectricPower      # NVCC_DRAM
    power_1v8  : ElectricPower      # SD/NAND IO option
    power_snvs : ElectricPower      # SNVS (3V)
    # power_vtt comes from a separate module wrapped around TPS51200, fed from power_1v35
```

## Reviewer findings (2026-05-04 first pass)

| # | Sev | Issue | Status |
|---|---|---|---|
| 1 | Critical | SNVS not first-on (sequence violation) | **FIXED** — 470 nF EN-delay caps on each buck (RC ≈ 4.7 ms) ensure SNVS LDO settles before any buck soft-starts |
| 2 | Critical | Coin-cell OR missing (lost SNVS on 5V drop) | **FIXED** — moved into PowerTree as `vbat_in` interface + 2× SS34 OR diodes; coin cell + 1k limit live in Debug feeding `vbat_in` |
| 3 | High | NVCC_DRAM ripple/setpoint margin | **PARTIAL** — added ferrite bead + 22 µF bulk on buck output; tightened setpoint to ±4%. FB ripple-injection network deferred to upstream PR on `atopile/ti-tps563201`. |
| 4 | High | Inductor 3.1 A < 5.5 A OCL peak | **FIXED** — `assert inductor.saturation_current >= 4.5A` on all 3 bucks; picker chose SXN SMNR5030-2R2N (4.5A) |
| 5 | High | Doc claimed SOT-23-5 but BOM is WSON-6 | **FIXED** — doc updated below |
| 6 | High | Shared VDDARM/VDDSOC, no bead | **FIXED** — split into `power_1v4_arm` and `power_1v4_soc` with ferrite bead between, local 10 µF + 100 nF on SoC side |
| 7 | Medium | No 5V inrush control | **DEFERRED** — owned by Power Input subsystem (load switch) |
| 8 | Medium | No PG; supervisor is TODO | **DEFERRED** — Debug subsystem (TPS3839 multi-input supervisor) |
| 9 | Low | FB divider current bound too loose | **DEFERRED** — needs upstream `atopile/ti-tps563201` PR |
| 10 | Low | X5R temp derating not in doc | **NOTED** below |

### #5 corrected: TLV75901 is WSON-6

Picker selects **TLV75901PDRVR** (WSON-6 with thermal EP), not SOT-23-5 as originally documented. WSON-6 with proper thermal-via array on EP gives θJA ≈ 45 °C/W. At 0.45 W worst case, ΔT ≈ 20 °C → die at ~90 °C with 70 °C ambient. **Layout requirement: ≥4 vias on EP to GND plane.**

### #10 X5R derating note

Picked output caps are 10 µF X5R 25V 0805. Combined derating:
- DC bias at 1.35V: ~−2% → 9.83 µF effective
- X5R at +85°C: ~−15% → 8.36 µF effective
- 3 caps in parallel = 25 µF worst-case (≈ 30 µF nominal)

Within TPS563201's 20–68 µF window, but at the lower edge. Adding the post-ferrite 22 µF bulk takes us to ~47 µF total — comfortably mid-window.

## Still open / TODOs

- [ ] `ato create part` for TPS51200 (VTT termination) — not in registry. → DDR3 subsystem.
- [ ] Add multi-input reset supervisor (TPS3839 watching 3V3 + 1V4 + 1V35) → Debug subsystem.
- [ ] Coin cell + 1 kΩ charge limit → Debug subsystem (feeds `psu.vbat_in`).
- [ ] Layout: TLV75901 EP needs ≥4 GND vias.
- [ ] Layout: ferrite bead placement at the "boundary" between buck and consumer pour.
- [ ] Upstream PR on `atopile/ti-tps563201`: add FB ripple-injection R/C option for all-ceramic outputs.
- [ ] Upstream PR on `atopile/ti-tps563201`: tighten `feedback_divider.current` to 50–200 µA range.
- [ ] Decide separate 3V3_ANALOG ferrite split for LAN8742A VDDA (Issue 5 from Ethernet review).
