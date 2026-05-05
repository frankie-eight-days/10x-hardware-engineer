# MPU subsystem (i.MX 6ULL)

## Role

The brain. Wraps the MCIMX6Y2CVM08AB BGA-289 package with all the rails, decoupling, crystals, and exposed buses for the rest of the board.

## Architecture

```
                  ┌─────────────────────────────────────────┐
                  │         i.MX 6ULL (MCIMX6Y2CVM08AB)     │
   power_3v3 ────▶│ VDD_HIGH_IN, NVCC_*, VDDA_ADC_3P3       │
   power_1v4_arm ▶│ VDDARM_IN                               │
   power_1v4_soc ▶│ VDDSOC_IN                               │
   power_1v35 ───▶│ NVCC_DRAM ×6                            │
   power_snvs ───▶│ VDD_SNVS_IN                             │
                  │                                          │
                  │ Internal caps: VDD_*_CAP × 6 (1 µF each)│
                  │                                          │
   24 MHz xtal ──▶│ XTALI/XTALO (CCM main clock)            │
   32.768k xtal ▶│ RTC_XTALI/RTC_XTALO (RTC for SNVS)      │
                  │                                          │
                  │ Exposed buses:                           │
                  │   ddr3_bus (50+ signals to DDR3 chip)   │
                  │   rmii, mdio, enet_reset → Network       │
                  │   usdhc1 → MicroSD                       │
                  │   usb_otg1 → USB-C #2                    │
                  │   uart2 → USBUart console                │
                  │   sai2 → Audio                           │
                  │   jtag, reset → Debug                    │
                  │   boot_mode0/1, heartbeat, sd_cd → Debug │
                  └─────────────────────────────────────────┘
```

## Pin/ball groups (from datasheet §3 Modules List)

### Power supplies (must all be connected)

| Rail | i.MX pin name | Source | Decoupling |
|---|---|---|---|
| 3V3 | VDD_HIGH_IN | power_3v3 | 1× 4.7µF + 1× 100nF |
| 3V3 | NVCC_GPIO, NVCC_NAND, NVCC_LCD, NVCC_CSI, NVCC_ENET, NVCC_SD1, NVCC_UART, NVCC_PLL | power_3v3 | 1× 100nF each |
| 3V3 | VDDA_ADC_3P3, ADC_VREFH | power_3v3 (with ferrite filter ideally) | 1× 100nF + 1× 1µF |
| 1.4V | VDDARM_IN | power_1v4_arm | 1× 100nF + 1× 10µF |
| 1.4V | VDDSOC_IN | power_1v4_soc | 1× 100nF + 1× 10µF |
| 1.35V | NVCC_DRAM ×6 (M6, L6, K6, J6, H6, G6) | power_1v35 | 6× 100nF + 1× 22µF bulk |
| 3V SNVS | VDD_SNVS_IN | power_snvs | 1× 100nF + 1× 1µF |
| Internal | VDD_ARM_CAP ×4, VDD_SOC_CAP, VDD_HIGH_CAP, VDD_SNVS_CAP, VDD_USB_CAP, NVCC_DRAM_2P5 | (output cap from internal LDO) | 1µF X7R 0402 each |

Total decoupling: ~25 caps. Heavy commonization: most are 100nF X7R 0402 (already common); the bulk 22µF for DDR is one new line item.

### Crystals

- **24 MHz (CCM main)**: `XTALI` / `XTALO`. 18–22 pF load caps. Per i.MX 6ULL HDG §"Clocking", load capacitance per leg ≈ `2·(CL - C_stray) ≈ 22 pF` for CL=12 pF crystals.
- **32.768 kHz (RTC)**: `RTC_XTALI` / `RTC_XTALO`. 12.5 pF crystal, with internal load caps in the SNVS domain — only the crystal is needed externally per HDG §"RTC".

### DDR3 bus (50+ signals, all dedicated balls)

Exposed as a `DDR3Bus` interface that the DDR3 module connects to:

- DRAM_ADDR[15:0] — 16 address lines
- DRAM_DATA[15:0] — 16-bit data
- DRAM_DQM[1:0], DRAM_SDQS[1:0]_P/N — byte-lane mask + diff strobes
- DRAM_SDCKE[1:0], DRAM_SDCLK0_P/N, DRAM_SDBA[2:0] — control + bank
- DRAM_RAS_B, DRAM_CAS_B, DRAM_SDWE_B — row/col/write enable
- DRAM_CS[1:0]_B, DRAM_ODT[1:0] — chip select + on-die termination
- DRAM_RESET — DDR3 reset
- DRAM_VREF — 0.5 × NVCC_DRAM reference (single-ended VTT/VDDQ ref)
- DRAM_ZQPAD — ZQ calibration (240 Ω resistor to ground)

For our single-chip 16-bit DDR3, only CS0/ODT0 and SDCKE0 are used. CS1/ODT1/SDCKE1 left floating.

### Direct-to-peripheral balls (no IOMUXC mux required)

| Peripheral | Balls |
|---|---|
| ENET1 RMII | ENET1_TX_CLK, ENET1_TX_EN, ENET1_TX_DATA0/1, ENET1_RX_EN, ENET1_RX_ER, ENET1_RX_DATA0/1 |
| USDHC1 | SD1_CLK, SD1_CMD, SD1_DATA0–3 |
| USB OTG1 | USB_OTG1_DP, USB_OTG1_DN, USB_OTG1_VBUS, USB_OTG1_CHD_B |
| UART2 (console) | UART2_TX_DATA, UART2_RX_DATA |
| JTAG | JTAG_TCK, JTAG_TMS, JTAG_TDI, JTAG_TDO, JTAG_TRST_B, JTAG_MOD |
| Boot mode | BOOT_MODE0, BOOT_MODE1 |
| Reset | POR_B |
| Power button | ONOFF (tied high via 100kΩ to VDD_SNVS to keep board on) |

### IOMUXC-muxed peripherals (TODO from pinmux research)

These need ball-to-alt-function decisions. Pending research agent results:

- **SAI2 I²S** (TX_BCLK, TX_SYNC, TX_DATA) → likely on CSI_DATA / LCD_DATA / NAND_DATA balls
- **MDIO/MDC** for ENET1 → likely on GPIO1_IO06/07
- **I²C1** (SCL, SDA) for any future expansion → likely on UART4_TX/RX

Once the research agent returns the canonical pinmux, fill in here.

## First-principles design notes

### Decoupling philosophy

i.MX 6ULL has dozens of power balls. Per the HDG §"Power Supply Decoupling": one **100 nF X7R 0402 per ball**, placed within 5 mm of the package. Plus **bulk capacitance per rail** (≥10 µF) within 30 mm. We commonize at the 100 nF and 10/22 µF level — that's ~25 individual 100 nF caps but only one BOM line.

Internal LDO caps (`VDD_*_CAP`): per datasheet §3.1.3, each gets a **1 µF X7R 0402** between the cap pin and GND. These are decoupling for the on-die regulators that produce VDD_HIGH (2.5V), VDD_SOC, VDD_ARM, etc. Not optional.

### DRAM_VREF: 0.5 × VDDQ via 1% divider

Datasheet §"DDR3L Interface" requires DRAM_VREF = NVCC_DRAM / 2 ± 1 %. Two **1 kΩ ±0.1 % resistors** in series from NVCC_DRAM to GND, midpoint to DRAM_VREF, with a **100 nF + 10 nF** filter cap. 1 % matched resistors hold tolerance.

### DRAM_ZQPAD: 240 Ω 1 % to GND

DDR3 ZQ calibration reference. **Single 240 Ω 1 % 0402** from DRAM_ZQPAD to GND. Sets the impedance for output drivers via the chip's internal calibration.

### ONOFF strap

Per datasheet, ONOFF must be tied to VDD_SNVS via 100 kΩ pull-up to keep the board in "on" state continuously. If we ever want a power-button feature, a SPST momentary to GND on this pin would toggle it. v0: just pull high.

### Boot mode wiring

BOOT_MODE0/1 from Debug subsystem (via 2-pos DIP). i.MX 6ULL boot ROM samples these at reset; default position selects "internal boot from BT_CFG straps." BT_CFG straps default to "probe USDHC1 → USDHC2 → NAND → eMMC" for SD-first boot — no extra straps needed for our case.

### POR_B handling

POR_B is the master reset. Routed to:
- Reset button (Debug)
- JTAG header pin 10 (nSRST)
- 10 kΩ pull-up to 3V3 (Debug-side)
- 100 nF debounce cap to GND (Debug-side)

### Unused peripheral straps

**TEST_MODE**: must be tied to GND (datasheet §"Special Signal Considerations"). Critical — board won't function if TEST_MODE floats high.

**Reserved/unused**: NGND_KEL0, GPANAIO, JTAG_MOD (typically pulled low), unused USB_OTG2, unused UART3-5, unused CSI_*, unused LCD_DATA*. Tie unused inputs to GND or 3V3 per HDG §"Recommended Connections for Unused Analog Interfaces."

For v0 we connect only the peripherals we use; unused balls are left floating in atopile (atopile WILL warn but won't fail). Layout-time we'll add tie-downs for the critical unused-input pads.

## Module API

```ato
module MPU:
    # Power rails
    power_3v3      : ElectricPower
    power_1v4_arm  : ElectricPower
    power_1v4_soc  : ElectricPower
    power_1v35     : ElectricPower
    power_snvs     : ElectricPower

    # DDR3 — exposed as bus
    ddr3           : DDR3Bus

    # Peripherals
    rmii           : RMII
    mdio           : MDIO
    enet_reset     : ElectricLogic       # MPU drives, active-low
    usdhc1_cmd     : ElectricLogic
    usdhc1_clk     : ElectricLogic
    usdhc1_dat     : ElectricLogic[4]
    sd_cd          : ElectricLogic
    usb_otg1       : USB2_0
    uart2          : UART_Base
    sai2           : I2S
    jtag           : JTAG
    reset          : ElectricLogic
    heartbeat      : ElectricLogic
    boot_mode0     : ElectricLogic
    boot_mode1     : ElectricLogic
```

## Open questions / TODOs

- [ ] Pinmux research agent — fill in IOMUXC alt-function choices for SAI2, MDIO, I²C.
- [ ] Add 1µF X7R 0402 decap on every VDD_*_CAP pin (24 internal LDO caps).
- [ ] DRAM_VREF divider + DRAM_ZQPAD resistor.
- [ ] TEST_MODE strap (must = GND).
- [ ] ONOFF 100kΩ to VDD_SNVS.
- [ ] 24 MHz crystal + load caps (22 pF).
- [ ] 32.768 kHz crystal (RTC).
- [ ] Decide if we want the 3V3_ANALOG ferrite split for VDDA_ADC_3P3 (Ethernet review flagged this for cleaner SI on PHY VDDA — same question here for ADC).
