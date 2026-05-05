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

3.2 W mono into 4 Ω, 1.8 W into 8 Ω (per datasheet @3.3V VDD). Plenty for a desktop speaker.

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

MAX98357A `nSD_MODE` pin selects:

| nSD_MODE voltage | Behaviour |
|---|---|
| 0 V | Shutdown (zero current) |
| 0.16 × VDD | Right channel only |
| 0.50 × VDD | Stereo (L+R) / 2 sum |
| 0.84 × VDD | Left channel only |
| VDD | Stereo (L+R) / 2 sum |

Tying `nSD_MODE` directly to VDD via **100 kΩ pull-up** gives "active, stereo sum" — simplest and matches what `aplay` outputs on the default ALSA path. (i.MX 6ULL SAI is stereo by default; we don't have a separate left/right speaker setup so a sum is what we want.)

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

### Output filter: ferrite + 1 nF, twice

Class-D switches at ~300 kHz. Without filtering, that energy radiates from the speaker leads as EMI. Datasheet §"Output Filtering" recommends **filterless mode** (no LC filter) for short speaker cables (<1 m), but recommends a small **ferrite + cap** snubber for longer cables.

For a dev board with 30 cm of speaker wire, **ferrite-bead + 1 nF cap** snubber is the right answer:
- ferrite 600 Ω @ 100 MHz × 2 (commonized with power-tree ferrites — same C1017 part)
- 1 nF C0G × 2 from each output post-ferrite to GND
- Damps the BTL switching node before it leaves the PCB

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

## Open questions / TODOs

- [ ] Speaker connector (2-pin header) lands in Debug subsystem.
- [ ] First-article: scope the OUTP/OUTN waveform to confirm filter is doing its job (good blog content).
- [ ] If we ever want stereo, use TAS5825 (more pins, I²C — defer to v2).
