# 10x Hardware Engineer

An experiment in agent-driven, full-stack open-source hardware: design a Linux-class dev board from datasheet to shell prompt with as much LLM automation as physically possible, then write up what worked and what didn't.

## The board

**Target MPU:** NXP i.MX 6ULL (MCIMX6Y2CVM08AB, ~$5–6 @ qty 10)
- Cortex-A7 @ 800 MHz, MAPBGA-289, 0.8 mm pitch (forgiving for an open-stack BGA fanout)
- External DDR3, 2× 10/100 Ethernet MAC, USB 2.0 OTG, I²S, UART, SPI, USDHC
- Mainline Linux + U-Boot, NXP Yocto BSP, Olimex OLinuXino open KiCad reference

**Peripherals (v1 spec):**
- 256–512 MB DDR3L (Micron / Nanya, 16-bit, JLC stock permitting)
- 1× RJ45 Ethernet (LAN8720A or KSZ8081 PHY) — talk to it over the home LAN
- USB 2.0 host/OTG (also doubles as i.MX serial-download recovery — unbrick without JTAG)
- microSD slot (primary boot)
- Optional QSPI / eMMC footprint (stretch)
- I²S Class-D audio amp (MAX98357A) + speaker pads — fun ngspice subject
- USB-UART bridge (CH340N, JLC Basic) for the shell, plus 4-pin TTL UART header
- 10-pin Cortex JTAG/SWD header (don't ship without it)
- PMIC: PCA9450 or PF3000 (TBD — pick what JLC has stocked)

## The stack

| Layer | Tool | Notes |
|---|---|---|
| Datasheet RAG | Docling → LanceDB hybrid → custom FastMCP | Required citations on every answer |
| Block diagrams | Mermaid (source-of-truth) + drawio MCP | mermaid diffs in git, drawio for blog |
| Schematic | atopile (`.ato`) | parametric `pick_part`, JLC-aware |
| Parts | jlcparts SQLite + easyeda2kicad | offline part DB + symbol/footprint pulls |
| Layout | KiCad 9 + Seeed kicad-mcp-server | 39-tool MCP |
| Routing | KiCad PNS (DDR3, USB, ENET) + Freerouting CLI (boring nets) | hybrid; agent does length-tune constraints |
| Sim | ngspice + InSpice | power rails, PMIC loop, Class-D filter |
| Fab/PCBA | JLCPCB API | basic/preferred parts where possible |
| Firmware build | Buildroot + mainline U-Boot + mainline Linux | Yocto is overkill |
| Firmware debug | OpenOCD + JTAG + serial-download recovery | |
| LLM | Opus 4.7 (1M) for layout/DDR; Sonnet 4.6 for cheap Q&A | prompt-cache datasheet + HDG |

## Phases

1. **Hardware capture** — block diagram → atopile modules → KiCad netlist → BOM
2. **Layout** — placement, BGA escape, DDR3 length-matching, Ethernet diff pairs, fab files
3. **Bring-up** — power-on, UART, U-Boot from SD
4. **Linux** — kernel + Buildroot rootfs, shell over UART
5. **Network** — DHCP, SSH from laptop
6. **Audio demo** — I²S out to the Class-D amp, play a sine wave from userland
7. **Blog** — honest write-up: what the agents did, what they got wrong, where humans had to step in

## Why this is supposed to be novel

Quilter shipped an i.MX 8M Mini DDR4 board with an autonomous physics engine in 40 hours — closed-source. atopile / tscircuit / Diode have shipped public LLM-driven schematics but only on 2-layer ESP32-class boards. **Nobody has publicly crossed the DDR Rubicon on a fully open agent stack.** That's the gap.

## Status

Scaffolding. See [PLAN.md](PLAN.md) for the decision log and open questions.

## License

MIT.
