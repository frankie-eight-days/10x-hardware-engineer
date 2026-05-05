# USB-C #2 — USB OTG1 connector

## Role

Second USB-C receptacle on the board, wired to the i.MX 6ULL's **USB OTG1**
peripheral. Acts as a USB **device** (UFP) by default — host PC enumerates
the board as a gadget (CDC-ACM serial, USB-Ethernet, mass storage, fastboot,
whatever the kernel exposes). USB-C #1 is power-only (HUSB238 PD), USB-C #3
is the CH340 console.

## Architecture

```
USB-C connector (USB2_0TypeCHorizontalConnector from atopile/usb-connectors)
  │
  ├── 5.1kΩ pulldowns on CC1 + CC2  → declares device-mode (UFP)
  ├── 1A polyfuse on VBUS           → fault protection
  ├── USBLC6-2SC6 ESD on D+/D-      → ±15kV cable-side protection
  │
  ├── VBUS (5V from cable) ─── 100Ω ─── i.MX USB_OTG1_VBUS  (detect ≥4.4V)
  ├── D+/D-                ────────── i.MX USB_OTG1_DP / DN
  └── GND ────────────────────────── system GND

i.MX OTG1 ID pin (GPIO1_IO00 ALT2 = ANATOP_OTG1_ID)
  └── 100kΩ pull-up to 3V3 (inside MPU module) → device-mode default
```

## Why UFP-only in v0

Full USB-C OTG with role detection requires a CC-monitoring chip
(TUSB320LAI, FUSB302, etc.). These add cost + pinmux + a Linux driver and
aren't needed for a dev board where the typical use case is:

> Plug board into laptop → board appears as `/dev/ttyACM0` and `usb0`
> network interface.

CC pulldowns alone are enough to advertise UFP behavior to a USB-C host,
which then injects 5V VBUS. The MPU sees VBUS ≥4.4V on USB_OTG1_VBUS and
the controller starts in device mode.

For host-mode operation in v2: replace this subsystem with one that
includes a TUSB320LAI driving USB_OTG1_ID.

## Why the 100Ω in series with VBUS detect

NXP AN12028 §"USB VBUS Detection" recommends a 100Ω current-limit resistor
in series with `USB_OTG1_VBUS`. The on-chip VBUS_VALID comparator threshold
is ~4.4V (range 4.0–4.8V) and the pin is rated 5.5V abs. The 100Ω limits
in-rush + ESD bleed-through without dropping VBUS below the threshold.

> **Earlier bug:** the original wiring used a 20kΩ/30kΩ divider that pulled
> VBUS down to 3V at the chip pin. With the threshold at 4.4V the chip
> would never see "VBUS present" and OTG1 would never enable. **Fixed.**

## Why ID is pulled HIGH

i.MX 6ULL ANATOP_OTG1_ID convention:
- ID **HIGH** (or floating, internal pull-up enabled) → device mode
- ID **LOW** (grounded) → host mode

100kΩ pull-up to 3V3 on GPIO1_IO00 inside the MPU module forces device
mode. Cheap, deterministic, won't fight any host that mistakenly drives ID.

## Module API

```ato
module USBOtg:
    power_5v_vbus = new ElectricPower    # 5V VBUS from host cable (after fuse)
    usb_dp_dn     = new DifferentialPair # to mpu.usb_otg1
```

VBUS is exposed because the MPU module needs it for VBUS_VALID detect.

## Open / TODOs

- [ ] **v2:** add TUSB320LAI for full USB-C OTG role detection.
- [ ] First-article: confirm host enumeration on a Linux laptop +
      confirm VBUS_VALID asserts within 100ms of cable insert (USB CDC spec).
- [ ] Layout: keep D+/D- diff-pair length matched to ±50 mil, 90Ω
      differential impedance.
