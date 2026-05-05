# DDR3 SDRAM subsystem

## Role

Main system memory for the i.MX 6ULL. Single-chip 16-bit DDR3L providing 256 MB at the MPU's MMDC interface. Required for U-Boot / Linux to load (i.MX boot ROM has only 128 KB ROM + 256 KB OCRAM).

## Architecture

```
i.MX 6ULL MMDC (MPU.ddr3 bus, 1.35V signaling)
  │
  ├── DRAM_ADDR[0..13] ──────── A0..A13          (14 row/col lines)
  ├── DRAM_SDBA[0..2] ────────── BA0, BA1, BA2    (3 bank select)
  ├── DRAM_DATA[0..7] ────────── DQL0..DQL7       (lower byte lane)
  ├── DRAM_DATA[8..15] ───────── DQU0..DQU7       (upper byte lane)
  ├── DRAM_DQM[0] ────────────── DML              (lower byte mask)
  ├── DRAM_DQM[1] ────────────── DMU              (upper byte mask)
  ├── DRAM_SDQS0_P/N ─────────── DQSL / DQSL#     (lower byte strobe pair)
  ├── DRAM_SDQS1_P/N ─────────── DQSU / DQSU#     (upper byte strobe pair)
  ├── DRAM_SDCLK0_P/N ────────── CK / CK#         (system clock pair)
  ├── DRAM_SDCKE0 ────────────── CKE              (clock enable)
  ├── DRAM_CS0_B ─────────────── CS#              (chip select)
  ├── DRAM_RAS_B / CAS_B ─────── RAS# / CAS#
  ├── DRAM_SDWE_B ────────────── WE#
  ├── DRAM_ODT0 ──────────────── ODT              (on-die termination ctrl)
  ├── DRAM_RESET ─────────────── RESET#
  └── DRAM_VREF (0.675V) ─────── VREFCA + VREFDQ  (shared, single divider)

External per Nanya datasheet:
  ZQ pad → 240Ω 1% → GND  (impedance calibration)
  VDD/VDDQ × ~12 pins → 12× 100nF X7R 0402 + 2× 10µF X5R 0805 bulk
```

## Part choice

| Role | MPN | LCSC | Stock |
|---|---|---|---|
| DDR3L SDRAM | Nanya NT5CC128M16JR-EK | C428583 | 79 ⚠️ low |

**Backup:** Micron MT41K128M16JT-125 (pin-compatible, more stock).

## Datasheet anchors (Nanya version 1.5 04/2019)

- **Page 1**: features, programmable functions, package summary
- **Page 5**: 96-ball TFBGA top-view ball map (used for atomic-part pinout)
- **Pages 6-7**: ball descriptions + power supply requirements
- **Pages 9-12**: power-on initialization sequence (200µs stable → RESET# released → 500µs → MR2/MR3/MR1/MR0 → ZQCL)

## First-principles design notes

### Why x16 single-chip instead of two x8

i.MX 6ULL MMDC supports 16-bit or 32-bit DDR3 buses. We use 16-bit. Two options:
- **One x16 chip** (this design): 2× fewer chips, fewer balls to fan out, simpler T-topology, 256 MB capacity.
- **Two x8 chips**: more capacity (512 MB) but doubles BGA fanout work, requires fly-by routing for clocks/address with proper termination, ×2 power, ×2 decoupling.

For a dev board where 256 MB is plenty for Linux + apps, **one x16 chip** is the right tradeoff. Industry convention for sub-$10 single-chip boards.

### Why DDR3L (1.35V) not DDR3 (1.5V)

- i.MX 6ULL natively supports both. NVCC_DRAM is the rail and we picked 1.35V from the buck because:
  - **Power**: 1.35V × draw is ~25% less than 1.5V × draw → less heat
  - **Datasheet compatibility**: NT5CC...JR-EK is dual-voltage capable (per page 1 footnote)
- Tradeoff: DDR3L is ~10% slower spec for the same speed grade because of lower drive amplitude. Not a problem at our DDR3-1600 target.

### Single VREF for both VREFCA and VREFDQ

JEDEC JESD79-3 allows VREFCA and VREFDQ to share one supply for "low noise" applications (i.e., single-chip). For multi-DRAM-rank designs you'd want separate dividers per chip for isolation, but with one chip we tie VREFCA + VREFDQ together to the existing 1k/1k divider already in the MPU module.

The MPU module's `package.DRAM_VREF` is the i.MX 6ULL's single Vref output which we drive into both VREFCA and VREFDQ on the DDR3 chip.

**TODO:** add `vref` field to `DDR3Bus` interface in mpu.ato + wire from MPU to DDR3 chip.

### ZQ calibration: 240Ω 1% to GND

Required by JEDEC for output driver impedance calibration. Single 240Ω 1% 0402 resistor from the chip's ZQ ball (L8) directly to GND. Place near the chip; don't share with anything else.

### Decoupling: ~12× 100nF per VDD/VDDQ ball + bulk

Looking at the 96-ball top-view ball map, count of supply pins:
- **VDD**: B2, D9, G8, K2, K8, N1, N9, R1, R9 = 9 pins
- **VDDQ**: A1, A8, C1, C9, D2, E9, F1, H2, H9, plus VSSQ-pairs = ~9 pins
- **VSS / VSSQ**: many

Datasheet implicit recommendation (typical for DDR3L x16): one 100nF X7R 0402 per ball, plus 2-3× 10µF X5R 0805 bulk close to the chip on the 1V35 rail. We commonize 100nF (already 20+ in BOM).

### RESET# handling

Active low. **Held below 0.2×VDD during power-on for 200µs minimum.** `DRAM_RESET` from the MPU drives this directly; the MPU's MMDC controller manages the init sequence in firmware. 100kΩ pull-down on RESET# is good practice (already in i.MX 6ULL by way of internal pulldown — confirm in HDG; if not, add external).

### Layout-time TODOs (not in atopile, in KiCad)

This module produces **netlist only**. The hard work happens in KiCad:

1. **Length-matching groups** per i.MX 6ULL HDG / AN12042 DDR3L Layout Guidelines:
   - **Address/control + clock** group: match within ±25 mil; CK pair to be the slowest
   - **Byte 0 group**: DQ0-7, DM0, DQS0/DQS0# match within ±25 mil; reference DQS0
   - **Byte 1 group**: DQ8-15, DM1, DQS1/DQS1# match within ±25 mil; reference DQS1
   - **Diff pairs**: CK/CK# and both DQS pairs intra-pair match ±5 mil
2. **Continuous reference plane** under DDR3 traces (no plane splits)
3. **Short, point-to-point** routing (T-topology not needed for single chip)
4. **Place chip ≤30 mm from MPU** to keep within timing budget
5. **Impedance**: 50Ω single-ended, 100Ω diff; rely on stackup target

## Module API

```ato
module DDR3:
    power_1v35 : ElectricPower    # NVCC_DRAM
    bus        : DDR3Bus          # imported from mpu.ato
    vref       : Electrical       # shared VREFCA + VREFDQ from MPU's divider
```

## Open questions / TODOs

- [x] Add `vref` to DDR3Bus interface in mpu.ato; wire `package.DRAM_VREF` to it. **DONE.**
- [x] Confirm Nanya NT5CC128M16JR EasyEDA symbol pin numbering. **Partially:** found the DQSL/DQSL# pair was swapped, fixed. Found 6 NC pads (J1, J9, L1, L9, M7, T7) missing from the auto-gen, added.
- [x] Prefix DDR3 address signals (`DDR_A0`, `DDR_A4`, etc.) to avoid name collisions with MPU BGA pad coordinates A4/A5/A6/A11/A13. **DONE.**
- [x] Resolve `min() iterable argument is empty` build crash. **DONE** — see "Build blocker" section above; root cause was atopile bbox bug on circle-only silkscreen, fixed by adding `fp_rect` body outline.
- [ ] Decide if we need a 100kΩ pulldown on DRAM_RESET externally.
- [ ] First-article: scope CK/CK# at the chip with a high-Z probe — confirm differential swing, no overshoot.
- [ ] Reviewer agent run after schematic is in place.
- [ ] File the `get_bbox_from_geos` Circle/Arc bug upstream at github.com/atopile/atopile.

## Build blocker — RESOLVED (upstream atopile bug)

The earlier `min() iterable argument is empty` crash during the PCB-write stage
turned out to be an **atopile bug**, not a schematic issue. Resolved 2026-05-05.

**Root cause:** `faebryk/exporters/pcb/kicad/transformer.py:get_bbox_from_geos`
has unimplemented `...` stubs for the `Circle` and `Arc` branches:

```python
elif isinstance(geo, kicad.pcb.Circle):
    # TODO: calculate extremes.extend([geo.center, geo.end])
    ...
```

When `set_designator_positions` runs on a *new* footprint (added in this build),
it computes the silkscreen bounding box. The Nanya footprint that `ato create
part` imported from EasyEDA has 96 `fp_circle` shapes on `F.SilkS` (one per
ball, decorative) and **zero** `fp_line`/`fp_rect`/`fp_arc`. So `extremes`
stays empty, and the chained `Geometry.bbox` call eventually does `min(...)`
on `[]`.

**Fix applied:** added a single `fp_rect` body outline to the Nanya .kicad_mod:

```kicad_mod
(fp_rect
    (start -3.75 -6.5)
    (end 3.75 6.5)
    (stroke (width 0.12) (type solid))
    (fill no)
    (layer "F.SilkS")
    (uuid "00000000-0000-0000-0000-000000000001")
)
```

Corners match the 7.5 × 13.0 mm package body from the filename
`TFBGA-96_L13.0-W7.5...`. This also gives the BGA a proper silkscreen body
outline on the assembled board — strictly an improvement over the 96 floating
circles.

**How the actual stack trace was found** (worth noting for future opaque
atopile errors): the `min()` message printed by `ato build` is just the
exception message; the full traceback is logged to
`~/Library/Logs/atopile/build_logs.db` table `logs`, column `python_traceback`
as JSON. Query with:

```bash
sqlite3 ~/Library/Logs/atopile/build_logs.db \
  "SELECT python_traceback FROM logs \
   WHERE message='min() iterable argument is empty' \
   ORDER BY id DESC LIMIT 1"
```

The earlier "Renaming net `D7`→…" diagnostic clue was a red herring — those
are normal info logs emitted as nets get processed; the crash happens later
in `update_pcb` → `set_designator_positions` → `get_bounding_box`, which has
nothing to do with net renaming.

**Should also be filed upstream:** atopile/faebryk should implement the Circle
and Arc bbox branches (the math is straightforward: for Circle, extents are
`(center.x ± radius, center.y ± radius)`).
