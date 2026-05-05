# Plan & Decision Log

Living document. Captures *why* each choice was made so future-me (or a future agent) can re-evaluate when something breaks.

## Thesis

Be the first public end-to-end open-stack agent flow that ships a DDR-class Linux dev board, all the way to a shell prompt over the network. Quilter has done DDR closed-source/physics-based; atopile/tscircuit/Diode have done open-source LLM but stayed on 2-layer ESP32 boards. The novel contribution is the open intersection.

## Key decisions

### MPU: NXP i.MX 6ULL
**Why:** 0.8 mm pitch BGA-289 is the most forgiving DDR-capable BGA, ~$5–6 at qty 10, best-in-class open-source ecosystem (Olimex, Variscite reference designs), DDR3 instead of LPDDR4 means 4-layer is feasible.
**Considered & rejected:**
- STM32MP135 — TFBGA-289 0.5 mm pitch, harder fanout, thinner LCSC stock
- Allwinner T113-S3 — QFP-128 is *too* easy; skipping BGA defeats the blog-post premise
- i.MX 8M Mini — LPDDR4 + 6–8 layer HDI puts it out of open-stack reach

### DDR: DDR3L 16-bit, single chip
**Why:** Single-chip DDR3 dodges fly-by topology length-matching across multiple devices. 16-bit (vs 32-bit) halves the byte-lane routing burden. DDR3L (1.35 V) matches i.MX 6ULL native rails.
**Open:** Exact part TBD — query jlcparts for in-stock 256/512 MB ×16 DDR3L.

### Ethernet PHY: LAN8720A (likely)
**Why:** RMII, 3.3 V, sub-$1, JLC-stocked, used in 80% of i.MX 6ULL reference designs. KSZ8081 is the alternative if LAN8720A is out.

### Audio: MAX98357A I²S Class-D
**Why:** 3.2 W mono, integrated DAC, no external clock, $2 on LCSC, datasheet is short enough to fit fully in context. Output filter is a great ngspice teaching example for the blog.

### USB-UART: CH340N onboard
**Why:** JLC Basic part, SOP-8, ~$0.30. Beats requiring a USB-TTL dongle. Backup: 4-pin 0.1" header on the same UART for native access.

### Power input: USB-C PD sink + bench supply, OR'd

**Two-source manual-select OR** — never both connected simultaneously, so two Schottkys are safe (no current-hogging concern).

```
USB-C ── HUSB238 (5V fixed) ── SS54 ─┐
                                     ├── 5V_BOARD → discrete buck/LDO chain
Bench ── P-FET (rev-pol) ──────── SS54 ─┘
```

USB-OTG VBUS is *not* part of this OR — host mode = we source it; device mode = it powers us, others off.

| Role | Pick | LCSC | JLC | Notes |
|---|---|---|---|---|
| USB-C 16-pin USB2.0 | Korean Hroparts TYPE-C-31-M-17 | C2765186 | Basic | Canonical JLC-friendly |
| USB-C PD sink (5V R-strap) | Hynetek HUSB238 | C2829277 | Extended ($3 fee) | No Basic alternative; CC1/CC2 5.1kΩ pulldowns mandatory |
| OR Schottky ×2, 5A | SS54 | C22452 | Basic | ~0.55V drop @ 5A; bucks see ~4.5V min, fine |
| P-FET reverse-pol | AO3401A | C15127 | Basic | $0.02, Rds(on) 50 mΩ, SOT-23 |
| Bench terminal block, 5mm 10A | Ningbo Kangnex EK500V-5.0-02P | C8270 | Basic | 2-pin green screw |

**Note:** initial C-numbers were unverified. Live JLC sweep done — see `boards/imx6ull-devboard/hardware/BOM.md` for the verified BoM with corrections (HUSB238 was wrong, terminal block was wrong) and stock/fee reality check.

### Discrete buck/LDO power tree from 5V_BOARD

PMIC choice: **none.** Discrete chain. Bench-supply spec (32V/3.2A) gives unlimited headroom; even if we go to a higher input later we don't change the topology.

| Rail | V | Approx I | Stage | Candidate | atopile pkg |
|---|---|---|---|---|---|
| `VDD_ARM_SOC` | 1.4V | ~600 mA | buck from 5V | TPS62933 / TPS563201 | `atopile/ti-tps563201` |
| `NVCC_DRAM` | 1.35V (DDR3L) | ~400 mA | buck | TPS563201 | `atopile/ti-tps563201` |
| `VTT_DRAM` | 0.675V | tracked | sink/source LDO | TPS51200 | `ato create part` |
| `VDD_3V3` | 3.3V | ~1 A | buck | TPS54560x | `atopile/ti-tps54560x` |
| `VDD_1V8` | 1.8V | ~300 mA | LDO from 3V3 | TLV75901 | `atopile/ti-tlv75901` |
| `VDD_SNVS` | 3V | ~1 mA | LDO + coin cell | LDK220 | `atopile/st-ldk220` |

**Sequencing:** i.MX 6ULL requires VDD_SOC_IN before VDD_ARM_IN before NVCC_DRAM (RM ch. "Power Management" / AN5187). Open-drain power-good daisy-chain across regulators is the discrete trick — each stage's PG enables the next.

### Boot: SD primary + USB serial-download recovery
**Why:** i.MX 6ULL boot ROM supports USB serial download via NXP `uuu` / `imx_usb_loader` — you can recover a bricked board over the same USB-OTG port without ever needing JTAG. SD is the working boot medium; eMMC/QSPI is a stretch goal.

### Layers: 4-layer (Sig / GND / PWR / Sig)
**Why:** Cheapest stackup that still gives DDR3 a continuous reference plane. 6-layer is the safer call; 4-layer is the riskier "can we?" answer that makes the blog post interesting.

### Build system: Buildroot, not Yocto
**Why:** Single defconfig file = LLM-friendly. Yocto's layer system is brutal to reason about. Yocto wins for production; Buildroot wins for one-off boards and learning.

### LLM: Opus 4.7 (1M ctx) + Sonnet 4.6
**Why:** Opus for layout review, DDR rule generation, datasheet synthesis where errors are expensive. Sonnet for cheap iterative Q&A during firmware bring-up. Prompt-cache the i.MX 6ULL Datasheet + HDG (~300k tokens) as always-on context; RAG the 5000-page Reference Manual.

## Phase plan

1. **Repo + RAG** — scaffold, ingest datasheets with Docling, stand up FastMCP datasheet server.
2. **Block diagram + atopile skeleton** — mermaid power tree + peripheral diagram, top-level `.ato` module.
3. **Schematic capture** — PMIC, DDR3, MPU, Ethernet, USB, audio, UART, SD, JTAG modules. Resolve all parts via `pick_part`.
4. **Layout** — placement, BGA escape, DDR3 length-matching constraints (agent generates KiCad custom rules), Ethernet diff pairs, Freerouting on remaining nets.
5. **DRC clean + fab** — Gerbers, BOM, CPL, JLCPCB API quote.
6. **Bring-up** — UART → U-Boot → SD boot → Linux mainline + Buildroot rootfs.
7. **Network** — SSH over LAN.
8. **Audio demo** — `aplay sine.wav`.
9. **Post-mortem & blog**.

## Open questions

- PMIC choice: PCA9450 (newer) vs PF3000 (more references). Pending JLC stock check.
- DDR3 vendor: Nanya is JLC-friendly, Micron has better open IBIS models. Probably Nanya.
- 4-layer vs 6-layer — defer until DDR3 routing constraints are written; if the agent can't get a clean reference plane on 4, escalate.
- Closed-loop sim → schematic iteration: aspirational. May ship as "agent reads ngspice output and proposes a cap value" without full closure.
- Whether to write our own KiCad MCP or use Seeed's. Default: use Seeed, fork if needed.

## Non-goals

- Production-grade certifications (FCC/CE).
- Yocto, real-time kernel, secure boot.
- DDR4 / LPDDR4. The point is to make DDR3 work *on an open stack*, not push the SI envelope.
- Pretty silkscreen art. The blog post is the deliverable; the board is the proof.
