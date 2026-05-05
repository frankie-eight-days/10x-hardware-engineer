# Power Input subsystem

## Role

Get 5V into the board, from either of two sources:

1. **USB-C #1 (PD sink)** via Hynetek HUSB238 — negotiates 5V from any USB-C source (PD-aware or default-Vsafe5V).
2. **Bench supply** via 2-pin terminal block, with reverse-polarity protection.

Both sources feed `5V_BOARD` through 5A Schottky diodes (manual-select OR — never both connected simultaneously).

## Architecture

```
USB-C #1 (TYPE-C-31-M-17, commonized)
    │ VBUS
    ├── HUSB238 [VIN]
    │      [VSET=GND] → request 5V default
    │      [ISET=GND] → request max current
    │      [CC1, CC2] → chip handles negotiation
    │      [GATE]     → unused in v0 (no external FET)
    │ VBUS                      ┐
    └────[SS54 Schottky D1]─────┤
                                 │
Bench input (DIBO 2-pin block)   │
    │  +              ┐          │
    └─[Source][AO3401A P-FET]    │
       │   [Drain]                │
       │      └──[SS54 D2]────────┤
       │                          │
       Vin- ───┬── Gate ─[10kΩ]──┘
               │
               GND ────────────── Board GND
                                  │
                                  ▼
                              5V_BOARD
                                  │
                            [bulk 100 µF + 10 µF + 100 nF]
```

## Part choices

| Role | Part | LCSC | JLC |
|---|---|---|---|
| USB-C connector | TYPE-C-31-M-17 (commonized) | C2765186 | Extended |
| PD sink controller | Hynetek HUSB238_002DD | C7471904 | Extended |
| Schottky 5A OR (×2) | MDD SS54 (commonized) | C22452 | Basic |
| P-FET reverse-polarity | AO3401A | C15127 | Basic |
| Terminal block 2-pin | DIBO DB910-6.35-2P-GN-S | C395872 | Extended |

## First-principles design notes

### Why HUSB238 (R-strap PD) over plain CC pulldowns

A USB-C device with just `5.1 kΩ pulldowns on CC1/CC2` will receive Vsafe5V from any compliant source — no PD negotiation. So why HUSB238?

- **PD sources hold their advertised 5V tighter** than raw Vsafe5V. With a PD contract, the source guarantees current capability (1.5A or 3A); without, it might fold back at 100 mA.
- **Future flexibility** — if we want to request 9V or 12V (e.g., to run a higher-current motor demo), we change the VSET strap resistor. No PCB respin.
- **Diagnostics** — HUSB238 has I²C registers exposing the negotiated contract; useful for blog content showing the agent reading state.

### VSET = GND for 5V request

HUSB238 datasheet table: VSET grounded directly = "request 5V default contract." Other configurations request 9/12/15/20V. We tie directly to GND.

### ISET = GND for max current

ISET pin sets the requested current. Tied to GND = "highest available." For 5V/3A the source will provide what it can. We're drawing far less than 3A so this is permissive.

### GATE pin: floating in v0

HUSB238 GATE drives an external N-FET that switches VBUS through to a downstream load only after PD negotiation succeeds. In v0 we leave it floating and use VIN directly via the Schottky — relying on USB-C's mandatory Vsafe5V default. **TODO v2**: add the AO3400A external switch FET for clean enable/disable.

### Schottky OR (no current-hogging)

Two SS54 5A Schottkys in OR. Manual-select (only one source plugged in at a time), so current-hogging via positive tempco is not a concern. ~0.55V drop @ 5A → board sees minimum 4.45V from a 5V source. All bucks/LDOs handle this comfortably.

### P-FET reverse polarity protection (bench input only)

User plugs bench supply, polarity not enforced by connector type. Reverse polarity is plausible.

**AO3401A topology:**
```
Vin+ ─[Drain]─[P-FET]─[Source]─ Vout+
                │
                └─ Gate ─[10kΩ]─ Vin- (board GND when correct polarity)
```

- **Correct polarity** (Vin+ positive): Source charges to Vin+, Gate stays at GND via R, Vgs = -Vin+ ≈ -5V → P-FET fully ON, ~50 mΩ Rds(on) → ~50 mV drop @ 1A.
- **Reverse polarity** (Vin+ negative): Source pulled negative, Gate stays at +Vin+ (now positive), Vgs = +5V → P-FET OFF. Body diode (drain→source) reverse-biased → blocks.

The 10 kΩ gate-to-GND resistor limits inrush gate current during transients.

### USB-C #1 D+/D- handling

USB-C #1 is power-only. D+/D- pins on the connector connect to HUSB238's D+/D- (used for BC1.2 detection if a legacy USB-A→USB-C adapter is in use). No board logic touches them. CC1/CC2 routing is to HUSB238 only.

### Bulk capacitance on 5V_BOARD

After the OR diodes, we add **100 µF + 10 µF + 100 nF** bulk on 5V_BOARD. Calc:

- TPS563201 startup inrush per buck: ~0.4 A for 1 ms = 0.4 mC charge each.
- Three bucks soft-starting nearly simultaneously: ~1.2 mC needed.
- A 100 µF cap supplies 1.2 mC at ΔV = Q/C = 1.2 mC / 100 µF = 12 mV — negligible droop.
- 10 µF MLCC for HF transient response, 100 nF for sub-µF noise.

### What the input subsystem does NOT do

Per the reviewer's note from Power Tree (Issue 7), inrush control during the *first* energization (when board input caps are at 0V) is NOT addressed by this module. A dedicated **inrush limiter** (NTC thermistor or MIC2026 load switch with Toff slew) would help if the bench supply trips on inrush. **TODO**: instrument first-power-on, decide if needed.

## Module API

```ato
module PowerInput:
    power_5v_out : ElectricPower    # 5V_BOARD downstream
```

(No external interfaces beyond the output rail — USB-C connectors and bench connector are integral.)

## Open questions / TODOs

- [ ] `ato create part` for AO3401A or use stdlib PFET — verify package="SOT-23" picks AO3401A automatically.
- [ ] ESD protection on USB-C #1 line side (TVS array on D+/D- + CC1/CC2).
- [ ] v2: add external switch FET on HUSB238 GATE for clean enable/disable.
- [ ] First-article: scope the bench-supply inrush. If >2 A, add NTC thermistor or load switch.
