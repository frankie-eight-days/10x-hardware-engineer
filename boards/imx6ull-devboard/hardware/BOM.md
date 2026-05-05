# Bill of Materials — i.MX 6ULL Dev Board v0

Verified against JLCPCB / LCSC May 2026 (see commit history for sweep date). **Re-verify Basic/Preferred status before order — JLC rotates these monthly.**

## Active components

| # | Role | MPN | LCSC | JLC | Stock | Qty-10 |
|---|---|---|---|---|---|---|
| 1 | MPU | NXP MCIMX6Y2CVM08AB | C414042 | Extended | 20,362 | $9.88 |
| 2 | DDR3L 2Gb ×16 | Nanya NT5CC128M16JR-EK | C428583 | Extended | **79** ⚠️ | $7.88 |
| 3 | Ethernet PHY 10/100 RMII | Microchip LAN8742A-CZ-TR | C621424 | Extended | 3,774 | $1.31 |
| 4 | RJ45 + magnetics | HanRun HR911105A | C12074 | Extended | 65,949 | $1.31 |
| 5 | USB-C 16-pin USB2.0 (×2) | TYPE-C 16PIN 2MD(073) SHOU HAN | C2765186 | Extended | 799,010 | $0.066 |
| 6 | USB-C PD sink | Hynetek HUSB238_002DD | **C7471904** | Extended | 9,191 | $0.39 |
| 7 | Schottky 5A OR (×2) | MDD SS54 | C22452 | **Basic** | 926,868 | $0.029 |
| 8 | P-FET SOT-23 (rev-pol) | AO3401A | C15127 | **Basic** | 1,366,313 | $0.047 |
| 9 | 5mm terminal block 2-pin | WJ128V-5.0-02P | ⚠️ C# unverified | likely Basic | — | ~$0.10 |
| 10 | Buck 3V3 / 1V4 / 1V35 (×3) | TI TPS563201DDCR | C116592 | Extended | 60,136 | $0.066 |
| 11 | DDR VTT term LDO | TI TPS51200DRCR | C34771 | Extended | **881** ⚠️ | $0.29 |
| 12 | LDO 1V8 300mA | TI TLV75901PDRVR | C544759 | Extended | 972 | ~$0.16 |
| 13 | RTC backup LDO | ST LDK220M-R | C443854 | Extended | 1,032 | $0.32 |
| 14 | I²S Class-D amp 3W | Maxim MAX98357AETE+T | C910544 | Extended | 11,595 | $1.17 |
| 15 | USB-UART bridge | WCH CH340N | C2977777 | Extended | 19,206 | $0.48 |
| 16 | microSD push-push | SOFNG TF-015 | C113206 | Extended | 13,387 | $0.14 |
| 17 | Cortex 10-pin 1.27mm hdr | Wcon 1310-1205G0M061CR02 | C721880 | unverified | — | — |
| 18 | CR2032 SMD holder | Q&J CR2032-BS-6-1 | C70377 | Extended | 26,926 | $0.13 |
| 19 | Reset tactile SMD | XKB TS-1187A-B-A-B | C318884 | **Basic** | 1,532,048 | $0.016 |
| 20 | 2-pos SMD DIP (BOOT_MODE) | kangshen DSIC02LSGET | C3293152 | unverified | — | — |

⚠️ = stock < 1000 or unverified — needs fallback or recheck before order.

## Backup substitutes (low-stock candidates)

| For | Substitute | LCSC | Reason |
|---|---|---|---|
| Nanya NT5CC128M16JR-EK | Micron MT41K128M16JT-125 | TBD | 79 units may not survive til order |
| TI TPS51200 | Silergy SY56012 | C155909 | 881 units |

## Assembly cost reality

- **3 of 20 lines are JLC Basic** (SS54, AO3401A, reset button). Everything else is Extended.
- **Extended setup fee:** ~$3 per unique part × ~17 = **~$51 in setup**.
- Unavoidable for a Linux-class board — no Basic version of MPU, PHY, PD sink, audio amp, USB-UART, etc.
- Total estimated PCBA setup cost (5 boards): **~$120–150** beyond per-board parts cost.

## Open verifications (do before submit)

1. **WJ128V-5.0-02P** — find correct C-number for the 2-pin variant. Original C8270 turned out to be 3-pin.
2. **Cortex JTAG header (#17)** — confirm JLC assembly assembles 1.27mm pitch SMD headers.
3. **DIP switch (#20)** — same.
4. **HUSB238 footprint** — C7471904 is the `_002DD` variant. Confirm the package matches what we'll generate via `ato create part`.
5. **Re-check JLC Basic/Preferred status** day-of-order — JLC rotates the list.

## Passives (TBD, will be solved at `ato build` time)

- Decoupling caps: 0402 X7R, 100 nF and 4.7 µF as needed
- Bulk caps: 47–100 µF / 10 V on 5V rail
- Power caps for buck regulators: per TPS563201 datasheet
- Pull-ups / pull-downs: 4.7 kΩ (I²C), 5.1 kΩ (USB-C CC), 12.1 kΩ (LAN8742A RBIAS), etc.
- LEDs: power-good, heartbeat, ETH link/act — generic 0603 red/green/blue
