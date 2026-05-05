# atopile knowledge pack — i.MX 6ULL board

Auto-loaded when working in this directory. **Project-specific layer** on top of the **official atopile skill** installed at `.claude/skills/ato/SKILL.md` (1505-line authoritative reference auto-loaded for the whole repo, installed via `npx ai-builder add skill atopile/ato`). This file adds i.MX 6ULL context that the official skill can't have. Defer to the official skill on syntax conflicts.

Authoritative source if the skill goes stale: https://raw.githubusercontent.com/atopile/atopile/main/.claude/skills/ato-language/SKILL.md

Current atopile: **v0.15.7 (April 2026)**. Docs URL slug is pinned to `atopile-0.14.x` even for newer point releases — this is correct, not stale.

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
