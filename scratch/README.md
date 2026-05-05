# scratch

Throwaway atopile experiments — not the real board. Used to validate the toolchain and try syntax.

## hello_resistor

Single 10 kΩ pull-down on a 3.3 V rail. Verifies parse → solve → pick LCSC part → emit KiCad PCB + BOM + datasheet.

```bash
cd hello_resistor/my_first_ato_project
ato build
```

Picked part on first run: UNI-ROYAL `0402WGF1002TCE` (LCSC `C25744`), a 10 kΩ ±1% 0402 resistor.
