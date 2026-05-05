# microSD subsystem

## Role

Primary boot medium. i.MX 6ULL boot ROM reads U-Boot from microSD when BOOT_MODE pins select SD. Linux rootfs lives on the same card after first boot.

## Architecture

```
i.MX 6ULL USDHC1 ── SD 4-bit interface ── SOFNG TF-015 push-push slot
                                          │
                                          ├── VDD = 3V3 (no UHS-I voltage switch for v0)
                                          ├── CMD pull-up: 10kΩ → 3V3
                                          ├── DAT[3:0] pull-ups: 47kΩ → 3V3
                                          └── CD switch → GPIO with 100kΩ pull-up
```

## Part choices

| Role | Part | LCSC | JLC |
|---|---|---|---|
| microSD push-push slot | SOFNG TF-015 | C113206 | Extended |
| Pull-up resistors | 10 kΩ + 47 kΩ + 100 kΩ (commonized values) | various | Basic |

## First-principles design notes

### Pull-up strategy

The SD spec (Physical Layer Spec §3.2 "Bus Pull-ups") specifies:
- **CMD**: 10–100 kΩ to VDD. CMD is open-collector during card-arbitration; the bus must default high.
- **DAT0**: pull-up needed (acts as busy signal in some modes); 10–100 kΩ.
- **DAT1, DAT2, DAT3**: pull-ups optional but recommended for clean startup; 10–100 kΩ.
- **CLK**: NO pull-up — host-driven push-pull only.

We use:
- **CMD: 10 kΩ** — strongest end of spec for fast command edges
- **DAT[3:0]: 47 kΩ** — light pull-up, lower steady current
- 5 pull-ups total = 5 resistors, but values commonize with rest of board (10k is everywhere; 47k is one new R value)

### CD (card detect) switch

Mechanical switch in the slot — shorts pin 9 to GND when a card is inserted. We pull pin 9 high through 100 kΩ to 3V3, route to MPU GPIO. Linux device-tree property `cd-gpios` maps this for hot-insert detection.

Some boards use the `DAT3` line as a CD signal (card-side internal pull-up makes DAT3 high without a card). We're not using that — explicit mechanical CD switch is more reliable.

### VDD supply: direct 3V3 (no load switch in v0)

Datasheet recommends a P-FET load switch on SD VDD so the host can power-cycle the card (for re-init or to enter sleep). For v0, **VDD tied directly to 3V3** — saves the FET, saves layout complexity, matches Olimex / NXP eval-board references.

If we ever need card-power-cycle for reliability, the load switch is a single SOT-23 P-FET drop-in. Mark as TODO for v2.

### No UHS-I (1.8V signaling)

UHS-I requires a USDHC voltage selector pin (`VSELECT`) and a buck or LDO that can swap NVCC_SD between 3.3V and 1.8V at runtime. Adds a regulator + level-shift logic. **Not worth it for v0**: U-Boot loads at 25 MHz / Linux boots fine at 50 MHz on 3.3V signaling, and SD cards' UHS-I gives at most 2× speed (104 MB/s vs 50 MB/s) which doesn't matter for boot.

NVCC_SD1 just stays on 3V3.

### Decoupling

100 nF + 10 µF on VDD — same caps already in our common parts list.

## Module API

```ato
module MicroSD:
    power_3v3 : ElectricPower      # in (VDD = 3V3)
    cmd        : ElectricLogic     # to MPU USDHC1_CMD
    clk        : ElectricLogic     # to MPU USDHC1_CLK
    dat        : ElectricLogic[4]  # to MPU USDHC1_DATA[3:0]
    cd         : ElectricLogic     # to MPU GPIO (card-detect)
```

## Open questions / TODOs

- [ ] Decide if we want a card-power-cycle FET. Skipped for v0.
- [ ] Verify SOFNG TF-015 footprint matches the JLC assembly stencil — push-push slots can have 0.5mm-pitch hold-down tabs that JLC might fight.
- [ ] When MPU pinmux lands, route USDHC1 to balls with NVCC_SD1 = 3V3 by default.
