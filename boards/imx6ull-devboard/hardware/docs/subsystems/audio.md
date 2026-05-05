# Audio subsystem

## Role

I²S Class-D audio output so the board can play sound. Used for:

- The "is it alive" demo: `aplay /usr/share/sounds/alsa/Front_Center.wav` after Linux boots
- ngspice simulation subject for the LC output filter (good blog material)
- Stretch: stream audio over Ethernet, demonstrate end-to-end pipeline

## Architecture

```
MPU SAI (I²S)        MAX98357A (TQFN-16, 3×3mm)         Class-D BTL output
                      ┌─────────────────┐
   BCLK   ─────────── BCLK              │
   LRCLK  ─────────── LRCLK         OUTP┼──[ferrite + 1nF]── speaker_p
   SDO    ─────────── DIN          OUTN┼──[ferrite + 1nF]── speaker_n
                      VDD (3V3)         │
                      GAIN ─[GND]       │  +9dB gain
                      nSD ─[100k→VDD]   │  active, stereo (L+R)/2 mix
                      └─────────────────┘
```

~1.6 W into 4 Ω, ~0.9 W into 8 Ω at VDD = 3.3 V (per datasheet Fig. 1). The 3.2 W / 1.8 W headline numbers in the datasheet are at VDD = 5 V. Plenty for a "hello world" demo speaker.

## Part choices

| Role | Part | LCSC | JLC |
|---|---|---|---|
| Class-D amp | Maxim MAX98357AETE+T (TQFN-16) | C910544 | Extended |
| Output ferrite ×2 | Sunlord GZ2012D601TF (commonized w/ power tree) | C1017 | Extended |
| Speaker connector | 2-pin 2.54 mm header (TBD in Debug subsystem) | — | Basic |

## First-principles design notes

### Why MAX98357A (not TPA3110 / TAS5825 / PAM8302)

- **MAX98357A** is the standard hobby/maker I²S Class-D amp because the datasheet is short (~30 pages), I²S is direct (no I²C config), and the gain/mode are set by single-resistor straps. $1.17 on JLC.
- TAS5825 needs I²C config — adds bring-up complexity. We want "boots, plays sound" with zero firmware fiddling.
- TPA3110 is analog-input only (no I²S). Would need a DAC. Costs more total.
- PAM8302 — analog-only, smaller. No I²S.

MAX98357A wins on simplicity for a "hello world audio" demo.

### Channel mode: stereo (L+R)/2 mix

MAX98357A `nSD_MODE` has a **100 kΩ internal pulldown**. Mode bands per datasheet (absolute voltage at the pin, not ratio):

| nSD_MODE voltage | Mode |
|---|---|
| < 0.16 V | Shutdown |
| 0.16 – 0.77 V | Stereo (L+R)/2 sum |
| 0.77 – 1.4 V | Right channel only |
| > 1.4 V | Left channel only |

To get **Stereo sum** with VDD = 3.3V, we need the pin between 0.16 V and 0.77 V. With the 100 kΩ internal pulldown, an external pull-up R sets:

```
V_SD = VDD × 100kΩ / (R_ext + 100kΩ)
```

For V_SD = 0.30 V (mid-band): R_ext = 1 MΩ. We use **1 MΩ pull-up to VDD**.

(Reviewer caught a bug here — the original 100 kΩ pull-up gave V_SD = 1.65 V which is "Left only", not "Stereo sum". The pin doesn't snap to "VDD = stereo sum" as some Adafruit-derived diagrams suggest; it actually selects "Left only" at full VDD because >1.4 V band wins.)

### Gain: +9 dB

`GAIN_SLOT` pin selects:

| GAIN_SLOT connection | Gain |
|---|---|
| 100 kΩ to GND | +9 dB |
| Float | +12 dB |
| 100 kΩ to VDD | +6 dB |
| GND direct | +15 dB |
| VDD direct | +3 dB |

**+9 dB** (100 kΩ to GND) is the default for "loud-but-not-clipping" desktop volume on a typical 4 Ω speaker. Driven by reasoning: 3.3V × 0.9 (Class-D efficiency) → ~3W into 4Ω at clip. With +9 dB and 1Vrms full-scale audio (typical line level), you reach clip near full ALSA volume — exactly the right operating point.

### Output: filterless mode

Datasheet §"Output Filtering" recommends **filterless mode** for short speaker cables (< 30 cm) — that's our case. OUTP/OUTN go straight to speaker pads.

Earlier draft used ferrite + 1 nF as a "Class-D filter," but the LC corner of 600 Ω@100 MHz / 1 nF lands at ~16 MHz — way above Class-D's 300 kHz switching fundamental. That's an EMI snubber, not a filter; mixing it with a real LC filter is explicitly warned against in the datasheet ("do not use the optional ferrite-bead filter in conjunction with the optional Class-D filter"). So: filterless mode, layout-time mitigations (short twisted-pair speaker wires, GND pour underneath).

If we ever need long speaker cables, drop in **10 µH + 470 pF** per output (proper LC, ~70 kHz corner). v2 if needed.

### VDD decoupling

Datasheet specifies **1 µF + 100 nF per VDD pin**. The MAX98357A has two VDD pins (7 and 8). Layout-wise these are usually shared — one 1 µF + one 100 nF close to the package, plus a 10 µF bulk on the rail.

Using 100 nF X7R 0402 and 1 µF X5R 0402 — both already in the board's common cap selection.

### Why 3.3V VDD (not 5V)

MAX98357A spec: 2.5–5.5V. At 5V VDD, max output power is higher (~5W). At 3.3V VDD, max is ~3.2W into 4Ω. We don't need >3W for a dev board demo, and using **3.3V** lets us share the rail with everything else — no separate audio rail, no extra ferrite bead.

If first-article reveals we want louder output, hopping to 5V VDD is a pour-and-paste change in layout.

### Speaker connector

Punted to Debug subsystem. Will use a **2-pin 2.54 mm header** to allow trivial speaker wire push-on, or just bare pads if soldering directly. No connector cost.

## Module API

```ato
module Audio:
    power_3v3  : ElectricPower      # in (Audio runs from 3V3)
    i2s        : I2S                # to MPU SAI
    speaker_p  : Electrical         # post-filter to speaker connector
    speaker_n  : Electrical
```

## Reviewer findings (batch review 2026-05-04)

| Sev | Issue | Status |
|---|---|---|
| Critical | nSD_MODE 100kΩ to VDD = "Left only" not stereo sum | **FIXED** — changed to 1MΩ |
| High | Ferrite + 1nF output is EMI snubber, not Class-D filter | **FIXED** — filterless mode |
| High | SD_MODE walks through modes during VDD ramp | **NOTED** — chip latches stable mode after VDD settles; if first-article shows the wrong mode at boot, drive nSD_MODE from MPU GPIO (TODO v2) |
| Medium | Doc claimed 3.2W into 4Ω at 3V3 — actually ~1.6W | **FIXED** in this doc |
| Medium | i.MX SAI must be master (MAX98357A is always slave) | Note: configure SAI as master in MPU pinmux + device-tree |
| Low | EP thermal pad — verify ≥4 vias in layout | Layout-time check |

## Open questions / TODOs

- [ ] Speaker connector (2-pin header) lands in Debug subsystem.
- [ ] First-article: scope the OUTP/OUTN waveform to confirm filterless EMI is acceptable.
- [ ] If first-article shows boot-time mode-walk problems on nSD_MODE, drive from MPU GPIO.
- [ ] If we ever want stereo, use TAS5825 (more pins, I²C — defer to v2).
