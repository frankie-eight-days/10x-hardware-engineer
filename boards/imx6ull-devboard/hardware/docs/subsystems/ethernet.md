# Ethernet subsystem

## Role

10/100 Mbps Ethernet so the board can join the home LAN, accept SSH, and serve TFTP/NFS during firmware bring-up.

## Architecture

```
MPU ENET1 (RMII) ── LAN8742A PHY ── magnetics (in RJ45) ── twisted pair
MPU MDIO ─────────── LAN8742A SMI
LAN8742A → 2× LED ── front panel link/activity
```

Single-chip 10/100 PHY, RMII (saves pins vs MII), no external switch. i.MX 6ULL has two ENET MAC cores; we use ENET1 only (ENET2 reserved for future / debug).

## Part choices

| Role | Part | LCSC | Cite |
|---|---|---|---|
| PHY | Microchip LAN8742A-CZ-TR | C621424 | LAN8742A datasheet |
| PHY package | atopile/microchip-lan8742a v0.3.0 | — | Pre-built atopile module — handles RBIAS, decoupling, crystal, straps. **Best canonical example in the registry.** |
| RJ45 + magnetics | HanRun HR911105A | C12074 | HR911105A datasheet |
| Crystal | (provided by atopile package) Yangxing X322525MRB4SI 25 MHz | (in pkg) | The atopile package bundles this. |

## First-principles design notes

### Why LAN8742A (not LAN8720A)

Both are pin-compatible 10/100 RMII PHYs. LAN8742A is **already packaged** in atopile (saves a multi-day `ato create part` for a 24-pin QFN), it has a built-in 1.2 V regulator (saves an external LDO), and it's stocked at JLC (3,774 units). LAN8720A would shave $0.30 off BOM but cost a week of effort. **Take the win.**

### Why single-chip, no external switch

Board is a dev/learning platform. Single port suffices for SSH + TFTP. A switch (e.g., RTL8305NB) would cost +1 IC, +N magnetics, +N RJ45s, +routing. Out of scope.

### RBIAS = 12.1 kΩ ±1%

Set by the LAN8742A datasheet as a precision bias for the internal current sources that drive the MDI pairs. Critical — wrong value = wrong line drive amplitude = link partner can't decode. Atopile package already encodes this.

### Crystal = 25 MHz, ⚠️ load caps are wrong upstream

LAN8742A wants 25.000 MHz ±50 ppm with crystal CL = 18 pF. The math for parallel-resonance load caps is:

```
CL = (Cload_left × Cload_right) / (Cload_left + Cload_right) + Cstray_per_leg
18 = Cload/2 + Cstray
Cload = 2 × (18 - Cstray) ≈ 24 to 30 pF (for Cstray = 3 to 6 pF/leg)
```

So each load cap should be **~27 pF**, not 18 pF.

**Bug in upstream `atopile/microchip-lan8742a@0.3.0`:** sets load caps to 18 pF (= the crystal's CL value, not the calculated load cap). This pulls the oscillator high by tens of ppm and risks blowing the ±50 ppm budget.

**Status:** can't override in our wrapper (atopile constraints are additive, not replacing). TODO: fork the package or file a PR upstream. For prototype boards we may live with a few extra ppm of clock offset; for production, fix.

### MDI termination strategy

LAN8742A drives current-mode into the magnetics. The package routes to `ethernet.pairs[0]` (RX) and `ethernet.pairs[1]` (TX) `DifferentialPair` — these go to the magnetics center taps + outputs. The center-tap caps (typically 2× 10 nF + 1 nF Bob Smith resistor network to chassis ground) are **not** in the LAN8742A package — we add them externally on the magnetics side. **TODO:** wire the magnetics + Bob Smith network in the Network wrapper.

### Bob Smith termination

For each unused pair (4-pin RJ45 in our case has 2 used, 2 unused) and the center-taps, the standard EMI termination is 75 Ω + 1 nF to chassis. Datasheet of HR911105A shows the recommended network. **TODO:** add when we wrap the RJ45.

### LED count

Two LEDs out of LAN8742A: link (ETH_LED1) + speed (ETH_LED2). Both routed to the RJ45 jack (HR911105A has integrated LEDs in the bezel — saves 2 board LEDs).

## Module API (provided to Board)

```ato
module Network:
    power_3v3 : ElectricPower      # rail in
    rmii       : RMII              # to MPU ENET1
    mdio       : MDIO              # to MPU
    reset      : ElectricLogic     # active-low, MPU drives
```

## Reviewer findings (2026-05-04 first pass)

Independent reviewer agent caught critical bugs in the upstream `atopile/microchip-lan8742a@0.3.0` package and our initial wrapper. Status:

| # | Severity | Issue | Status |
|---|---|---|---|
| 1 | **Critical** | nINTSEL strap missing pulldown — PHY latches nINTSEL=1, no REF_CLK to MAC | **FIXED** in `network.ato` (4.75kΩ pulldown) |
| 2 | **Critical** | MDI termination missing — 4× 49.9Ω TXP/TXN/RXP/RXN to VDDA per Microchip Checklist | **FIXED** in `network.ato` |
| 3 | High | HR911105A magnetics + Bob Smith + Y-cap to chassis | TODO — wrap when we add magjack |
| 4 | High | Crystal load caps wrong upstream (18pF should be ~27pF) | **DEFERRED** — can't override in wrapper. File upstream PR. |
| 5 | Medium | VDDA ferrite-bead split + 22nF RXCT cap | TODO — same wrapper |
| 6 | Medium | LED1/LED2 wiring polarity affects REGOFF strap | TODO — handle in Debug subsystem |
| 7 | Medium | i.MX RMII pad pull-config at boot must be high-Z (else fights MODE straps) | Note for MPU module |
| 8 | Medium | ESD protection on RJ45 cable side | TODO — same wrapper |
| 9 | Low | MDC pull-up unnecessary — push-pull MAC-driven | **FIXED** (removed) |
| 10 | None | Reset signal verified OK | — |

## Other open TODOs

- [ ] HR911105A magjack: build the wrapper (Bob Smith + ESD + bezel LEDs)
- [ ] File upstream PR on atopile/microchip-lan8742a for crystal cap value + missing termination
- [ ] Confirm i.MX 6ULL ENET1 RMII pad PUE=0 at reset in MPU pinmux
