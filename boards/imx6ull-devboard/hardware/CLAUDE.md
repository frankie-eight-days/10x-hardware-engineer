# atopile knowledge pack — i.MX 6ULL board

Auto-loaded when working in this directory. **Project-specific layer** on top of the **official atopile skill** installed at `.claude/skills/ato/SKILL.md` (1505-line authoritative reference auto-loaded for the whole repo, installed via `npx ai-builder add skill atopile/ato`). This file adds i.MX 6ULL context that the official skill can't have. Defer to the official skill on syntax conflicts.

Authoritative source if the skill goes stale: https://raw.githubusercontent.com/atopile/atopile/main/.claude/skills/ato-language/SKILL.md

Current atopile: **v0.15.7 (April 2026)**. Docs URL slug is pinned to `atopile-0.14.x` even for newer point releases — this is correct, not stale.

---

## Project state (2026-05-05)

**Build status:** ✓ `ato build` succeeds with **all 11 subsystems**. 66-line BOM, KiCad PCB generated.

**Subsystems shipped (11/11):**

| Subsystem | File | Notes |
|---|---|---|
| Power Input | `src/power_input.ato` | USB-C + bench OR'd via 2× SS54 + AO3401A rev-pol |
| Power Tree | `src/power_tree.ato` | 3× LocalBuck (forked) + LDO chain — see `src/parts/local_buck/` |
| Ethernet | `src/network.ato` | LAN8742A wrapper + HR911105A magjack + Bob Smith network + LEDs |
| USB-UART | `src/usb_uart.ato` | CH340N → USB-C #3 → MPU UART2 |
| USB-C OTG | `src/usb_otg.ato` | USB-C #2 → MPU OTG1, UFP/device default, 100Ω VBUS series |
| Audio | `src/audio.ato` | MAX98357A I²S (1MΩ pull-up for stereo-sum mode) |
| microSD | `src/microsd.ato` | SOFNG TF-015, 22µF bulk, 4-bit @ 3V3 |
| Debug | `src/debug.ato` | JTAG header + reset + boot DIP + LEDs + coin cell |
| MPU | `src/mpu.ato` | i.MX 6ULL pinmux + real crystals (24 MHz YXC + 32.768 kHz Epson) + I²C1 on UART4 pads |
| DDR3 | `src/ddr3.ato` | Nanya NT5CC128M16JR-EK 256 MB; needed silkscreen rect added to `.kicad_mod` (atopile bbox bug — see gotcha #7) |
| RTC | `src/rtc.ato` | NXP PCF8563T I²C RTC, battery-backed via VDD_SNVS, with own Epson 32.768 kHz xtal |

**Top-level wiring** in `src/main.ato:Board`. Rails have `override_net_name` (VCC_5V, VCC_3V3, etc.) — required for atopile's PCB writer not to crash on auto-name collisions.

## Known atopile gotchas (hit in this project)

1. **PCB-write segfault on duplicate net names.** Symptoms: `Build failed with code -11` after `Updating PCB` stage with warnings like `KiCad net 'cathode'/'hv' is already bound to another faebryk net`. Fix: set `override_net_name` on every top-level rail in `Board`.

2. **`AdjustableRegulator` picker contradiction with many consumers.** Symptoms: `Contradiction: No candidate assignment satisfies the discrete problem system` on `feedback_divider.chain.resistors[0/1].resistance`. Cause: `feedback_divider.v_in` bound to `power_out.voltage` over-constrains when many consumers add assertions. Fix: **fork the regulator package locally** (drop `from AdjustableRegulator` inheritance, hand-wire FB resistors). See `src/parts/local_buck/local_buck.ato` for the canonical fix.

3. **EasyEDA-imported BGA parts with malformed pin numbers.** Symptoms: warnings like `Could not match lead D18 to pad`, build fails at pinout stage. Cause: `ato create part` import drops BGA row letters on T/U-row pins (e.g., `BOOT_MODE1 ~ pin 10` instead of `pin U10`). Fix: read the actual datasheet's contact assignment table and `sed`-fix the .ato file. The i.MX 6ULL fixes are in `src/parts/NXP_Semicon_MCIMX6Y2CVM08AB/`.

4. **NC pads must be declared.** If footprint has 96 pads but .ato declares 90 signal/pin lines, atopile crashes during PCB write. Add `signal NC1 ~ pin J1` etc. for each unconnected pad to keep the count balanced.

5. **`+/- 50ppm` syntax not accepted.** For Crystal frequency tolerance, use `+/- 1%` or `+/- 0.005%` instead — atopile's number parser doesn't recognize `ppm`.

6. **`pin` is a reserved keyword** for child variable names. Don't name a sub-module `pin`. Use `pin_in` or similar.

7. **Silkscreen-only-Circle footprints crash `set_designator_positions`.** Symptoms: `min() iterable argument is empty` during `Updating PCB` stage on a freshly-added BGA/QFP whose `.kicad_mod` has only `fp_circle` shapes on `F.SilkS` (typical of EasyEDA-imported BGA pin-1 dot patterns). Cause: upstream atopile bug — `faebryk/exporters/pcb/kicad/transformer.py:get_bbox_from_geos` has a TODO `...` stub for the `Circle` branch (also for `Arc`), so silk-extreme list is empty, then `Geometry.bbox` calls `min()` on it. Fix: edit the `.kicad_mod` and add an `fp_rect` body outline on F.SilkS so the bbox has actual extents. Hit this with the Nanya DDR3 (96 circles, zero rects). The diagnostic clue (net renames before crash) is a red herring — those are just info logs that happen first; the crash is in `update_pcb` → `set_designator_positions` → `get_bounding_box`.

## DDR3 build blocker — RESOLVED

**The "blocker" was upstream atopile bug #7 above** (silkscreen-only-circles in Nanya's auto-imported footprint). Fix applied: `fp_rect` body outline added at `src/parts/Nanya_Tech_NT5CC128M16JR_EK/TFBGA-96_*.kicad_mod` (corners ±3.75 × ±6.5 mm matching the 7.5×13.0mm body). DDR3 module is now live in `main.ato` and builds clean.

Stack trace was hidden in `~/Library/Logs/atopile/build_logs.db` table `logs` column `python_traceback` — query with:
```bash
sqlite3 ~/Library/Logs/atopile/build_logs.db \
  "SELECT python_traceback FROM logs WHERE message='min() iterable argument is empty' ORDER BY id DESC LIMIT 1"
```
Returns JSON with full frames + locals — that's where the actual line number lives even when `ato build` only prints the bare error message.

(Earlier debug-context section preserved below for reference.)

## ⚠️ DDR3 BUILD BLOCKER — context for resume (HISTORICAL — see resolution above)

**Symptom:** `ato build` fails with single-line error `min() iterable argument is empty` during `Updating PCB` stage, **as soon as the DDR3 chip is instantiated**, even with zero connections to it.

**What's been tried (all failed identically):**

- Full DDR3 wiring (address + data + control + power + ZQ + decoupling)
- Comment out ZQ resistor + decoupling caps
- Comment out `vref` interface field on DDR3Bus
- Power-only connections (just VDD/VSS)
- **Zero connections — just `chip = new Nanya_..._package` in module body**
- Added 6 NC pad declarations (J1, J9, L1, L9, M7, T7) that auto-gen had skipped
- Prefixed DDR3 address signals with `DDR_` (DDR_A0, DDR_A4, etc.) to avoid collision with MPU's BGA pad coordinates A4/A5/A6/A11/A13

**Diagnostic clue (atopile log DB at `~/Library/Logs/atopile/build_logs.db`):**

```
Renaming net `D7`->`mpu.package-D7`
Renaming net `A7`->`mpu.package-A7`
Renaming net `A6`->`mpu.package-A6`
Renaming net `A5`->`mpu.package-A5`
Renaming net `A13`->`mpu.package-A13`
Renaming net `A11`->`mpu.package-A11`
[immediately followed by]
min() iterable argument is empty
```

atopile is renaming MPU package nets named after BGA coordinates when DDR3 enters the design. Something in that rename path leaves a net with empty pad list. Read atopile's KiCad PCB writer to find where `min()` over an empty list happens.

**Source paths to investigate:**

- atopile install: `~/.local/share/uv/tools/atopile/lib/python3.14/site-packages/atopile/`
- faebryk install: `~/.local/share/uv/tools/atopile/lib/python3.14/site-packages/faebryk/`
- Likely culprits: `atopile/exporters/pcb/`, `faebryk/libs/kicad/`, anything that calls `min()` with pad-position arguments
- `~/Library/Logs/atopile/build_logs.db` — full build log SQLite, query with `sqlite3` for context

**Possibly-fruitful next steps:**

1. **Get an actual stack trace.** `ato build` swallows the exception. Try `python -m atopile build` or set `ATOPILE_LOG_LEVEL=DEBUG`, or wrap in a Python `try/except` to print the traceback. The error site location is the key missing piece.

2. **Hand-roll the Nanya atomic part.** Auto-gen `ato create part --search C428583` may have a subtle structural issue we haven't identified. Write the .ato + .kicad_mod by hand from datasheet page 5 ball map. Use the LAN8742A package as the structural template — that one builds clean.

3. **Try a different DDR3 chip.** Micron MT41K128M16JT-125 is pin-compatible. Maybe its EasyEDA symbol is cleaner.

4. **File a minimal repro upstream.** The crash is reproducible with just two BGA chips that have overlapping coordinate-named nets. This is an atopile bug.

**What's confirmed correct (don't redo):**

- Datasheet pinout mapping in `src/ddr3.ato` (verified against Nanya v1.5 datasheet pages 4-7)
- DQSL pair fix (was swapped in auto-gen, now G3=DQSL, F3=DQSL#)
- 6 NC pads added (J1, J9, L1, L9, M7, T7)
- Address signal prefixing to DDR_A0, DDR_A4, etc.
- VREFCA + VREFDQ both share `bus.vref` (JEDEC-allowed for single-chip)
- ZQ via 240Ω 1% to GND
- 12× 100nF + 2× 10µF decoupling (commonized with rest of board)

**Files to look at when resuming DDR3:**

- `src/parts/Nanya_Tech_NT5CC128M16JR_EK/Nanya_Tech_NT5CC128M16JR_EK.ato` — atomic part (with sed-applied fixes)
- `src/parts/Nanya_Tech_NT5CC128M16JR_EK/TFBGA-96_*.kicad_mod` — footprint (96 pads)
- `src/ddr3.ato` — wrapper module (89 lines, schematic-complete)
- `src/main.ato` — DDR3 instantiation commented out near the end
- `docs/subsystems/ddr3.md` — full design doc + investigation log

## Quick commands

```bash
# Build (always from this hardware/ directory)
ato build

# Inspect log database for crash context
sqlite3 ~/Library/Logs/atopile/build_logs.db \
  "SELECT message FROM logs ORDER BY rowid DESC LIMIT 100"

# Create a part from LCSC C-number (interactive crashes — use --search)
ato create part --search "C428583" --accept-single

# Add a registry package
ato add atopile/<package-name>

# Verify pin labels match footprint pads
grep -oE "pin [A-Z]+[0-9]+" src/parts/<part>/<part>.ato | sort -u | wc -l
grep -oE '\(pad "[A-Z]+[0-9]+"' src/parts/<part>/<part>.kicad_mod | sort -u | wc -l
```

## Stdlib gotchas

- `LED` has `.diode.anode` / `.diode.cathode`, NOT `.anode` / `.cathode` directly. Bridge via the LED itself: `power.hv ~> r ~> led ~> power.lv`.
- `Inductor.max_current` doesn't exist — use `saturation_current`.
- `LEDIndicator` import name is wrong; use `LED` + `Resistor` directly.
- pin-headers package exposes `pins[N]` (Electrical[]), not `unnamed[N]`.
- Don't name variables `pin` (reserved keyword).
- Use `signal NAME ~ pin <coord>` syntax in atomic parts; bare `pin <coord>` lines are anonymous and break access from outside.

---

## Mental model

atopile is a **declarative, constraint-based DSL**. No control flow, no mutation, no execution order. You declare *what* the circuit is; the compiler + solver resolve it. Statements within a block are **order-independent** — assigning `r.resistance` twice is a constraint conflict, not "last write wins."

Three block types:

| Keyword | Purpose |
|---|---|
| `module` | Subsystem with children + connections |
| `interface` | Connectable boundary (buses, rails) — appears on both sides of `~` |
| `component` | Physical part with footprint |

Inheritance: `module Child from Parent:`. Empty body: `pass`.

---

## Connection operators (memorize)

| Op | Meaning | Use when |
|---|---|---|
| `~` | Wire — two interfaces are the same net. Bidirectional. Types must match. | `power_3v3 ~ phy.power_3v3`, `i2c ~ sensor.i2c` |
| `~>` | Bridge — insert a component in series. Component needs `can_bridge` trait. **Requires** `#pragma experiment("BRIDGE_CONNECT")`. | `power.hv ~> cap ~> power.lv`, `power_5v ~> ldo ~> power_3v3` |
| `->` | Retype — replace inherited child with a more specific type. | `regulator -> TI_TLV75901` |

`~` and `~>` are not interchangeable. Default to `~` for buses/rails. Use `~>` only when you literally want a 2-terminal device (R, C, L, LDO, fuse) inserted in line.

---

## Pragmas (top of every file that uses them)

```ato
#pragma experiment("BRIDGE_CONNECT")     # ~>
#pragma experiment("FOR_LOOP")           # for x in arr:
#pragma experiment("TRAITS")             # trait keyword
#pragma experiment("MODULE_TEMPLATING")  # new Foo<param=value>
#pragma experiment("INSTANCE_TRAITS")    # traits on instances
```

Using gated syntax without its pragma = compile error.

---

## Values: units + tolerances

```
3.3V                 # exact — usually wrong, picker matches nothing
10kohm +/- 5%        # bilateral — preferred for passives
3.0V to 3.6V         # bounded range
```

Units: `V`/`mV`, `A`/`mA`/`uA`, `ohm`/`kohm`/`Mohm`, `F`/`uF`/`nF`/`pF`, `H`/`uH`/`nH`, `Hz`/`kHz`/`MHz`/`GHz`, `s`/`ms`/`us`, `W`/`mW`. No space between number and unit.

**Critical:** zero-tolerance values (`r.resistance = 10kohm`) match no real parts. Always add `+/- N%`.

Assertions are constraints the solver must satisfy:

```ato
assert power.voltage within 3.0V to 3.6V
assert i2c.frequency <= 400kHz
assert sensor.i2c.address is 0x48
```

---

## Built-in interfaces (most-used)

| Type | Fields |
|---|---|
| `Electrical` | single bare node |
| `ElectricPower` | `.hv`, `.lv` (Electrical); `.voltage`, `.max_current` |
| `ElectricLogic` | `.line` (Electrical), `.reference` (ElectricPower) |
| `ElectricSignal` | analog variant of ElectricLogic |
| `DifferentialPair` | `.p`, `.n` (each ElectricLogic) |
| `I2C` | `.scl`, `.sda`; `.frequency`, `.address` (carries `requires_pulls`) |
| `SPI` / `MultiSPI` | `.sclk`, `.mosi`, `.miso`; `.frequency` |
| `UART` / `UART_Base` | `.tx`, `.rx` (+ flow control); `.baud_rate` |
| `I2S` | `.sck`, `.sd`, `.ws` |
| `USB2_0` / `USB3` / `USB_C` | bus power + diff pair |
| `Ethernet` | `.pairs[0..3]` (DifferentialPair), `.led_link`, `.led_speed` |
| `JTAG` | `.tck`, `.tms`, `.tdi`, `.tdo`, `.n_trst`, `.n_reset`, `.dbgrq`, `.vtref` (Power) |
| `SWD` | `.clk`, `.dio`, `.swo`, `.reset` |

**Every `ElectricLogic` needs `.reference` connected** to its power rail or solver throws floating-domain errors. Either wire each `signal.reference ~ power_3v3` or attach `has_single_electric_reference_shared` trait to the parent module.

---

## Imports

```ato
import Resistor                    # stdlib (bare name)
import ElectricPower, I2C          # NO — one per line
from "atopile/microchip-lan8742a/microchip-lan8742a.ato" import Microchip_LAN8742A
from "./local_module.ato" import MyThing
```

---

## Project structure

```
project/
├── ato.yaml                # manifest
├── src/                    # *.ato source
├── layout/                 # KiCad project (.kicad_pro, .kicad_pcb)
├── parts/                  # custom atomic parts + .kicad_mod
├── build/                  # generated outputs (gitignored)
└── .ato/modules/           # synced packages (gitignored)
```

`ato.yaml`:
```yaml
requires-atopile: ^0.14.0
paths:
  src: ./src
  layout: ./layout
builds:
  default:
    entry: main.ato:Board       # file:Module
dependencies:
  - atopile/microchip-lan8742a
```

---

## CLI (verified against installed v0.15.7)

```bash
ato create project --path <dir>  # scaffold new project (creates ato.yaml + main.ato)
ato create part                  # interactive: pull from LCSC/JLCPCB → parts/
ato create package               # scaffold a publishable package
ato create build-target          # add a new build entry to ato.yaml
ato add atopile/<pkg>            # add registry dep
ato sync                         # install/update deps from ato.yaml
ato remove <pkg>                 # remove a dep
ato dependencies <subcmd>        # full dep management
ato build                        # compile → KiCad PCB + BOM + datasheets + power tree
ato validate                     # syntax / consistency check
ato inspect <ref> --context <c>  # show what's connected at a node
ato view                         # block diagram / schematic viewer (browser)
ato serve                        # dev server
ato auth                         # registry auth
ato --non-interactive <cmd>      # CI-friendly
ato -v <cmd>                     # verbose (repeatable: -vv)
```

**Build artifacts** land in `build/builds/<target>/`:
- `default.bom.csv` / `.json` — BOM with picked LCSC parts
- `default.kicad_pcb` is in `layouts/<target>/` (preserved across rebuilds)
- `power_tree.md` — auto-generated mermaid power tree
- `default.variables.ato.json` — solver-resolved values
- `build/cache/*.pdf` — auto-downloaded datasheets for picked parts

`ato build` preserves manual KiCad placement/routing across rebuilds via stable hash-derived component IDs. There is no separate `ato install`, `ato test`, `ato format`, or `ato lint` — use `sync`, `validate`, and `build`-time assertions instead.

---

## What atopile CANNOT do — don't waste tokens trying

- **No length matching, impedance, stackup, or trace-width rules.** All hand-done in KiCad. Critical for DDR3 — atopile is netlist-only.
- **No SPICE / simulation.** Solver is unit/range constraints only.
- **No autorouter** exposed to `.ato`.
- **No control flow.** No `if`, `while`, functions, exceptions. Only `for x in arr:` (pragma-gated).
- **No state machines / firmware.** Hardware DSL only.
- **No first-class schematic authoring.** `.ato` is the source-of-truth; KiCad sch is generated/optional.

For DDR3, USB diff pairs, RMII, I2S clocks: declare nets correctly, hand off geometry to KiCad.

---

## Project-specific context

**Target:** NXP i.MX 6ULL Linux dev board with DDR3, Ethernet, USB, I²S audio, SD boot, JTAG.

**Packages that EXIST in the registry and we should use:**
- `atopile/microchip-lan8742a` — Ethernet PHY (RMII). Substitute for LAN8720A; canonical example file.
- `atopile/rj45-connectors`, `atopile/usb-connectors`, `atopile/programming-headers`, `atopile/pin-headers`, `atopile/testpoints`, `atopile/buttons`, `atopile/indicator-leds`, `atopile/mounting-holes`, `atopile/netties`
- LDOs/buck: `atopile/ti-tlv75901`, `atopile/ti-tps563201`, `atopile/ti-tps54560x`, `atopile/st-ldk220`
- I²C primitives: `atopile/microchip-24lc32` (EEPROM), `atopile/nxp-pcf8574/8575`, `atopile/ti-tca9548a`

**Packages that DO NOT exist — we must `ato create part` for these:**
- **NXP i.MX 6ULL** (MCIMX6Y2CVM08AB) — the chip itself
- **DDR3L SDRAM** (×16, e.g. Nanya / Micron MT41K)
- **PMIC** (PCA9450 / PF3000 / PF1550 — TBD)
- **MAX98357A** I²S Class-D audio amp
- **CH340N** USB-UART bridge
- **microSD socket** (Molex 502774 / Hirose DM3)

**Workflow for missing parts:** `ato create part --lcsc Cxxxxxx` (or `--mpn`) → generates `parts/<MFR>_<PN>/<PN>.ato` exposing every pin as raw Electrical. Then **wrap that package** in a high-level `module` exposing typed interfaces (`ElectricPower`, `I2C`, custom `interface`s like `RMII`/`DDR3Bus`). Use the LAN8742A package as the canonical reference: https://github.com/atopile/packages/tree/main/packages/microchip-lan8742a

---

## Common pitfalls (saved you the bug hunt)

1. **Zero tolerance** → picker fails. `r.resistance = 10kohm +/- 5%` always.
2. **Forgot pragma** → `~>` or `for` look like syntax errors. Add `#pragma experiment("BRIDGE_CONNECT")` etc. at top.
3. **Floating `ElectricLogic.reference`** → solver errors. Wire every `signal.reference ~ power_rail`.
4. **`~` type mismatch** → can't `~` an `I2C` to a `Power`. Match exactly.
5. **`required = True` not set on exposed interfaces** → board ships with floating rails, solver doesn't complain. Mark it on every externally-required interface.
6. **Two assignments to same param** → constraint conflict, not overwrite. Pick one.
7. **`ato install` after `ato add`** → some flows need `ato install` before `ato build` sees the dep.
8. **DRC excluded by default in package builds.** For the real board, do NOT inherit `exclude_checks: ["PCB.requires_drc_check"]`. Re-enable DRC.
9. **`.kicad_pcb` rewritten in place** by `ato build`. Commit layout before pulling new ato changes.
10. **Net names auto-hashed.** Force readable names with `iface.override_net_name = "VCC_3V3"`.
11. **`mpn` typos** → exact-match picker returns nothing. Pin with `lcsc_id` if unsure.
12. **I²C carries `requires_pulls`**, but it just *flags* the need — you still add the pull-up resistors yourself.

---

## Idiomatic patterns (copy these)

### Decoupling array

```ato
#pragma experiment("BRIDGE_CONNECT")
#pragma experiment("FOR_LOOP")
import Capacitor

decoupling = new Capacitor[4]
for c in decoupling:
    c.capacitance = 100nF +/- 20%
    c.package = "0402"
    power_3v3.hv ~> c ~> power_3v3.lv
```

### LDO in line

```ato
power_5v ~> ldo ~> power_3v3        # bridge through; LDO has can_bridge
ldo.v_in  = 5V   +/- 5%
ldo.v_out = 3.3V +/- 3%
```

### I²C pull-ups

```ato
pu_scl = new Resistor; pu_scl.resistance = 4.7kohm +/- 5%
pu_sda = new Resistor; pu_sda.resistance = 4.7kohm +/- 5%
i2c.scl.line ~> pu_scl ~> power_3v3.hv
i2c.sda.line ~> pu_sda ~> power_3v3.hv
i2c.scl.reference ~ power_3v3
i2c.sda.reference ~ power_3v3
```

### Pin a specific LCSC part

```ato
u1 = new MyChip
u1.lcsc_id = "C7426"
# OR
u1.mpn = "MAX98357AETE+T"
u1.manufacturer = "Maxim Integrated"
```

### Force a readable net name

```ato
power_3v3.hv.override_net_name = "VCC_3V3"
```

---

## When stuck — escape hatches

1. **`ato create part`** — interactive LCSC/JLCPCB pull. Generates raw-pin atomic part.
2. **Custom KiCad footprint** — drop `.kicad_mod` + STEP into `parts/MyPart/`, declare `component` with `pin VCC` etc. and traits `is_atomic_part`, `has_designator_prefix<prefix="U">`.
3. **Inherit from stdlib abstract** (`from LDO`, `from Regulator`, `from EEPROM`) — get the typed interface contract for free.
4. **The LAN8742A package** is the single best reference for "complex IC with custom buses + decoupling + pull-ups." Read it before writing any new chip wrapper: https://github.com/atopile/packages/blob/main/packages/microchip-lan8742a/microchip-lan8742a.ato

---

## Resources

- Official Claude skill (most concise authoritative reference): https://raw.githubusercontent.com/atopile/atopile/main/.claude/skills/ato-language/SKILL.md
- Docs index: https://docs.atopile.io/llms.txt
- Language essentials: https://docs.atopile.io/atopile/essentials/1-the-ato-language
- Compiler & CLI: https://docs.atopile.io/atopile/essentials/2-the-ato-compiler
- API reference: https://docs.atopile.io/atopile/api-reference/
- Package registry: https://packages.atopile.io · source: https://github.com/atopile/packages
- Examples: https://github.com/atopile/atopile/tree/main/examples
- LAN8742A canonical example: https://github.com/atopile/packages/tree/main/packages/microchip-lan8742a
- Discord: https://discord.gg/CRe5xaDBr3
