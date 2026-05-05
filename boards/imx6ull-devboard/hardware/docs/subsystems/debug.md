# Debug subsystem

## Role

Bring-up + dev workflow: poke at the board with JTAG, reset it, change boot mode, see what's happening via LEDs, keep the RTC alive over a power cycle, and break out the speaker + UART headers promised by Audio + USB-UART.

## Bag-of-stuff list

| Element | Component | LCSC | Why |
|---|---|---|---|
| JTAG/SWD 10-pin Cortex | Wcon 1310-1205G0M061CR02 (2×5 1.27mm SMD) | C721880 | Standard ARM debug header |
| Reset button | XKB TS-1187A-B-A-B (SMD tactile) | C318884 | JLC Basic, $0.016 |
| BOOT_MODE DIP | kangshen DSIC02LSGET (2-pos SMD) | C3293152 | Toggle SD ↔ USB serial-download |
| Status LEDs | stdlib `LEDIndicator` (2× 0603) | picker | Power good + heartbeat |
| CR2032 holder | Q&J CR2032-BS-6-1 (SMD) | C70377 | RTC backup battery |
| Charge-limit R | 1 kΩ stdlib | picker | Limits charge into non-rechargeable cell |
| OR Schottky | SS54 (commonized) | C22452 | Combined w/ LDK220 in PowerTree.vbat_in |
| 4-pin UART TTL header | atopile/pin-headers Male_2_54mm_1x4P_TH | (in pkg) | Console fallback |
| 2-pin speaker header | atopile/pin-headers Male_2_54mm_1x2P_TH | (in pkg) | MAX98357A output |

## First-principles design notes

### Reset button + RC debounce

i.MX 6ULL `POR_B` has internal pull-up + Schmitt input — minimal external help needed. We add:
- 10 kΩ external pull-up to 3V3 (deterministic deassertion)
- 100 nF debounce cap to GND (filters bounce on tactile button press)
- Tactile switch shorts POR_B to GND when pressed

### Cortex 10-pin pinout (industry standard)

```
 1 VTREF (3V3)     2 SWDIO/TMS
 3 GND             4 SWCLK/TCK
 5 GND             6 SWO/TDO
 7 KEY (nc)        8 NC/TDI
 9 GNDDetect      10 nSRST
```

i.MX 6ULL JTAG balls: `JTAG_TCK`, `JTAG_TMS`, `JTAG_TDO`, `JTAG_TDI`, `JTAG_TRSTB`, `POR_B`. We map:
- TMS → pin 2
- TCK → pin 4
- TDO → pin 6
- TDI → pin 8
- nSRST → pin 10 (also tied to POR_B / reset button net)
- 3V3 → pin 1 (target reference)
- GND → pins 3, 5, 9

`JTAG_TRSTB` is sometimes wired to pin 10 too (instead of nSRST), but most ARM probes drive nSRST. We tie `JTAG_TRSTB` to a 10 kΩ pull-up at the MPU (handled in MPU module) and don't expose it on the connector.

### BOOT_MODE strap

i.MX 6ULL boot ROM reads `BOOT_MODE[1:0]` at reset:
- `00` Internal fuses (production)
- `01` Serial download (USB-OTG / UART1) — recovery!
- `10` Internal boot (boot from BOOT_CFG straps — microSD in our case)

For dev work we want to toggle between **`10` (internal boot from microSD)** and **`01` (serial download recovery)**.

Implementation: BOOT_MODE0 = 1 (always), BOOT_MODE1 toggled by DIP switch:
- DIP closed → BOOT_MODE1 = 1 → `11` reserved (won't use)
- DIP open → BOOT_MODE1 = 0 → `01` (serial download)

Hmm — that's wrong. Need:
- `10`: BOOT_MODE1 = 1, BOOT_MODE0 = 0 → DIP1 high, DIP2 low
- `01`: BOOT_MODE1 = 0, BOOT_MODE0 = 1 → DIP1 low, DIP2 high

Two DIP switches naturally drive both bits. Using the kangshen DSIC02LSGET 2-position SMD DIP gives us 2× SPST. Each DIP-switch contact pulls its boot-mode signal to GND when CLOSED; default state via 10 kΩ pull-up to 3V3 = HIGH when OPEN.

Net mapping:
| DIP1 | DIP2 | BOOT_MODE1 | BOOT_MODE0 | Boot source |
|---|---|---|---|---|
| Open | Open | 1 | 1 | Reserved (avoid) |
| Closed | Open | 0 | 1 | Serial download (USB recovery) |
| Open | Closed | 1 | 0 | Internal boot (microSD) — **default** |
| Closed | Closed | 0 | 0 | Fuses (production) |

Default position label on silkscreen: "DIP1 OFF, DIP2 ON = SD boot."

### Status LEDs

Two onboard. More clutter than helpful.

- **3V3 PWR**: `LEDIndicator` directly on 3V3 rail. Stays on whenever board is powered.
- **HEARTBEAT**: Driven by an MPU GPIO; firmware pulses it once per second so you can see "yep, kernel is alive."

LED color: green for both (cheap, common, JLC Basic 0603 LEDs of green type are ~$0.005). Series resistor sized for 2 mA at 3V3 (low-current visible LED): R = (3.3 - 2.0) / 2e-3 = 650 Ω → use 680 Ω from common values.

`LEDIndicator` stdlib handles this — pass logic_in, power_in, plus optional active-low / use-MOSFET flags.

### Coin-cell + Schottky for VBAT_SNVS

PowerTree exposes `vbat_in: ElectricPower` for the SNVS coin-cell input. We feed it from:

- **CR2032 holder (Q&J CR2032-BS-6-1)** — 3V nominal, 220 mAh capacity.
- **1 kΩ charge-limit resistor** between cell + and the OR Schottky. CR2032 is a NON-rechargeable cell — the resistor protects against the i.MX SNVS rail back-driving current into it (max <100 µA reverse, well below the cell's tolerance for short pulses but still a no-no for long-term).
- **Schottky cathode at vbat_in.hv**, anode at the resistor. (Actually the OR is in PowerTree; we just feed `vbat_in` here with the cell + resistor + clean voltage.)

Net: Cell+ → 1 kΩ → vbat_in.hv (PowerTree handles the OR with LDK220 output).

When external power is on: SNVS rail = LDK220 output ≈ 3.0 V → Schottky path from cell is reverse-biased (cell at 3V, vbat_in at 3V → no current).

When external power off: LDK220 output drops to 0V → cell forward-biases its Schottky → vbat_in stays at 3V - 0.3V (Schottky drop) ≈ 2.7V which is still in SNVS spec.

### TTL UART + speaker headers

- 4-pin 2.54 mm: GND / 3V3 / TX (i.MX→header) / RX (header→i.MX), labeled on silkscreen.
- 2-pin 2.54 mm: speaker_p / speaker_n.

Both from the `atopile/pin-headers` package — saves us creating parts.

## Module API

```ato
module Debug:
    power_3v3        : ElectricPower
    power_5v         : ElectricPower    # for VBAT path? no — VBAT comes from cell directly
    vbat_to_psu      : ElectricPower    # → PowerTree.vbat_in

    # JTAG/SWD interface
    jtag             : JTAG             # to MPU JTAG pads

    # System
    reset            : ElectricLogic    # bidirectional: button or external probe → POR_B
    heartbeat        : ElectricLogic    # MPU GPIO → LED
    boot_mode0       : ElectricLogic    # to MPU BOOT_MODE0
    boot_mode1       : ElectricLogic    # to MPU BOOT_MODE1

    # Headers
    uart_console     : UART_Base        # 4-pin TTL header to MPU UART2 (parallel with CH340N)
    speaker_p        : Electrical
    speaker_n        : Electrical
```

## Open questions / TODOs

- [ ] Decide if we want a multi-input reset supervisor (TPS3839K33 or TLV803S) — defer to v2; rely on i.MX internal POR for v0.
- [ ] If first-article shows boot-mode confusion, swap DIP for a 4-pos rotary (boot-source selector with explicit positions).
- [ ] Layout: keep coin-cell trace short (low impedance for surge events).
- [ ] BOOT_CFG straps for SD slot configuration land in MPU module (need 6 strap resistors per i.MX 6ULL HDG §"Boot Configuration").
