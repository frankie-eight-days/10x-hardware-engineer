# USB-UART subsystem

## Role

USB-to-UART bridge so the host PC sees a `/dev/ttyUSB0` (Linux) / `/dev/cu.usbserial-*` (macOS) / `COMx` (Windows) when the dev board's console USB-C is plugged in. Used for:

- U-Boot console during first power-on
- Linux serial console (UART1 → CH340N → host)
- Bare-metal debug before USB stack is up

Belt-and-suspenders: the same MPU UART1 lines also go to a 4-pin TTL header for users who prefer their own FT232/CP2102 dongle, or for boot-time scope/logic-analyzer probing.

## Architecture

```
       USB-C #3 (TYPE-C-31-M-17, same part as #1 and #2, commonization)
            │ D+/D-/VBUS/GND/CC1/CC2
            ▼
  ┌─────────────────────┐
  │ CH340N (SOP-8)      │
  │ powered from 5V     │
  │ V3 internal LDO     │
  │ (100 nF decap)      │
  │                     │
  │ TXD ── MPU UART1_RX │   (3.3 V CMOS levels)
  │ RXD ── MPU UART1_TX │
  └─────────────────────┘
            │
            └── 4-pin header (3V3, GND, TX, RX) for USB-TTL fallback
```

CC1/CC2 of USB-C #3 carry 5.1 kΩ pulldowns (handled by the `usb-connectors` package — board acts as USB device).

## Part choices

| Role | Part | LCSC | JLC |
|---|---|---|---|
| USB-UART bridge | WCH CH340N | C2977777 | Extended ($3) |
| USB-C receptacle | TYPE-C-31-M-17 | C2765186 | Extended (already in BoM 3×) |
| Header | 4-pin 2.54 mm | generic | Basic |

## First-principles design notes

### Why CH340N (not FT232 / CP2102)

- **CH340N (SOP-8)**: $0.48, ~14k stock, includes integrated USB termination resistors, no crystal needed (built-in oscillator). 8-pin package = minimum BOM and minimum routing.
- FT232RL: more capable (modem control, CBUS) but ~$3, significantly higher cost. We only need TX/RX.
- CP2102N: similar capabilities to CH340N, comparable price, but worse JLC stock.

CH340N also has Linux/macOS/Windows host drivers in-tree (since kernel 4.18, since macOS 10.15, since Windows 10), no driver install needed today.

### Why power CH340N from 5V_BOARD (not VBUS)

Two options:

| Source | Pro | Con |
|---|---|---|
| VBUS (USB host) | Zero quiescent draw on board 5V when console unplugged | Backfeed risk on TX/RX when CH340N is off — TX into unpowered chip clamps via ESD diode, source current undefined |
| 5V_BOARD | Simple, deterministic, no backfeed | ~5–10 mA quiescent always |

5V_BOARD wins on simplicity. 5–10 mA on a dev board pulled by a bench supply is invisible. **Decision: 5V_BOARD.**

VBUS is left floating (only used by the connector itself).

### V3 pin decoupling

CH340N has an internal 3.3V LDO (V3 pin output). Datasheet requires a **100 nF X7R decap** from V3 to GND. V3 is not exposed externally — it powers the internal logic and IO. We just place the cap.

### TX/RX routing

- CH340N **TXD** is its UART output — connects to MPU **UART1_RX** (i.MX 6ULL receives from CH340N's transmit).
- CH340N **RXD** is its UART input — connects to MPU **UART1_TX**.
- This mirror is the easy thing to get wrong; cite name explicitly.
- 3.3V CMOS levels both sides (CH340N IO sourced from internal 3V3, i.MX 6ULL NVCC_NAND on UART1 pads is 3V3).

### nRTS unused (no flow control)

For `getty` console use, no flow control needed. nRTS pin left floating. (Datasheet says no internal pull on nRTS — it's an output, will idle high when no flow control is active. Floating is safe.)

### Series resistors on TX/RX (defensive)

Add 33 Ω series on each line. Two reasons:
1. Edge-rate slowdown reduces EMI from sharp 3.3V CMOS edges over the few cm of trace.
2. Current limit if the user accidentally plugs in a 5V-tolerant FT232 dongle to the TTL header *while* CH340N is also driving — the 33Ω limits short-circuit current to ~100 mA, well under both chips' 50 mA per-pin spec.

### TTL header pinout

Standard FTDI-cable order: `GND, CTS, VCC, TX (host→DUT), RX (DUT→host), nRTS`. We're not using flow control, so 4-pin minimum: `GND, 3V3, TX_DUT_OUT, RX_DUT_IN`. Mark silkscreen carefully because pinout is not standardized across boards.

**Decision: 4-pin 2.54 mm header, GND / 3V3 / TX (i.MX→header) / RX (header→i.MX).** No flow control. Ferrite optional (skip for v0).

## Module API

```ato
module USBUart:
    power_5v   : ElectricPower      # in
    power_3v3  : ElectricPower      # for TTL header power pin
    uart       : UART_Base          # to MPU UART1 (tx, rx only)
```

## Open questions / TODOs

- [ ] Confirm 33 Ω series doesn't slow USB-12-Mbps edges enough to matter (it shouldn't; USB stays on the D+/D- side of the chip, not the UART side).
- [ ] Decide TTL header polarity printing on silkscreen.
- [ ] ESD protection on USB-C #3 — same TVS array as the other USB-Cs (deferred to USB connector wrapper).
