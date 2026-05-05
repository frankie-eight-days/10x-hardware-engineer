# Subsystem design docs

One file per subsystem in the system block diagram. Each documents:

- **Role** in the board
- **Parts chosen** with LCSC + datasheet citations
- **First-principles reasoning** behind values that aren't directly from datasheets
- **Open questions / TODOs**

Living docs — update as the design evolves. Treat this as the engineering notebook that will feed the blog post.

## Files

- `ethernet.md` — RMII PHY + RJ45 + magnetics
- `power_input.md` — USB-C PD sink + bench supply OR + reverse polarity
- `power_tree.md` — discrete buck/LDO chain from 5V_BOARD
- `usb_uart.md` — CH340N USB-UART bridge for console
- `audio.md` — MAX98357A I²S Class-D amplifier
- `microsd.md` — boot SD slot
- `debug.md` — JTAG header, reset, boot DIP, status LEDs, coin cell
- `mpu.md` — i.MX 6ULL pin mapping + decoupling + reference connections
- `ddr3.md` — SDRAM single-chip, byte-lane wiring, layout requirements
