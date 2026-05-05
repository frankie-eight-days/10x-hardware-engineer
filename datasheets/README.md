# Datasheets

PDFs are git-ignored (too big). Fetch with `scripts/fetch_datasheets.sh` (TODO) or by hand.

## Acquired

| File | Pages | Source | Notes |
|---|---|---|---|
| `raw/imx6ull-MCIMX6Y2CVM08AB-datasheet.pdf` | 138 | bundled with reference design | NXP i.MX 6ULLIEC rev 1.2 (2017). Full electricals + ordering + DDR/MMDC + boot config. Authoritative. |
| `raw/imx6ull-hardware-guide.pdf` | 12 | nxp.com (AN5121) | i.MX 6ULL Hardware Design Checklist — short. |
| `raw/ddr3-nanya-NT5CC128M16JR.pdf` | — | bundled | Nanya 2 Gb DDR3-1600 ×16. **Probable DDR3 choice.** |
| `raw/ref-bundle/NT5CC256M16{DP,EP}.pdf` | — | bundled | Nanya 4 Gb DDR3 alternates. |
| `raw/emmc-MKEMB008GT1E.pdf` | — | bundled | 8 GB eMMC (stretch goal). |
| `raw/imx6ull-core-board-reference-schematic.pdf` | — | bundled | Full schematic of `yu-mingfu/IMX6ULL_Core_Board` — DDR3 byte-lane wiring, power tree, eMMC, connector. **Crib heavily.** |

## Gated behind NXP login (TODO when user is around)

- **i.MX 6ULL Reference Manual (`IMX6ULLRM.pdf`)** — ~5000 pages, register-level. Critical for firmware bring-up. Get from https://www.nxp.com/products/i.MX6ULL → Documentation → Reference Manual after sign-in.
- **i.MX 6ULL Hardware Developer's Guide (`IMX6ULLHDG.pdf`)** — DDR3/USB/Ethernet routing rules, layout recommendations. Different (longer) doc than AN5121.
- **AN5198** — i.MX 6UL/6ULL DDR Calibration.
- **AN12042** — i.MX 6UL/6ULL DDR3L Layout Guidelines.
- **AN5187** — Power Consumption.

## Pending fetch (public)

- Datasheets for our peripheral choices once finalized:
  - PMIC: PCA9450 / PF3000 / PF1550
  - Ethernet PHY: LAN8720A or LAN8742A
  - I²S amp: MAX98357A
  - USB-UART: CH340N
  - microSD socket: Hirose DM3 / Molex 502774

## Next step (RAG ingest)

Run **Docling** on every PDF in `raw/` → `parsed/` markdown + figure PNGs + per-page citations. Index with LanceDB hybrid (BM25 + dense embeddings). Serve via custom FastMCP at `mcp/datasheet-server/` exposing `lookup_register`, `lookup_pin`, `get_electrical_limit`, `get_routing_rule`. See repo `PLAN.md` for the full RAG strategy.
