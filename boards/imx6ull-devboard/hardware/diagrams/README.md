# Diagrams

Mermaid is the source-of-truth (text, diffs cleanly with `.ato`). Render via:

- **VS Code:** Markdown Preview Mermaid extension renders inline.
- **CLI:** `npx -p @mermaid-js/mermaid-cli mmdc -i system.mmd -o system.svg`
- **Quick view:** paste into https://mermaid.live
- **Drawio (presentation):** drawio MCP `open_drawio_mermaid` tool

## Files

- `system.mmd` — top-level block diagram. MPU + DDR3 + Ethernet + USB + audio + debug + power input.
- `power_tree.mmd` — focused view of the discrete buck/LDO chain with currents and sequencing notes.

## Notes

- **Two USB-C connectors** — one power-only (CC1/CC2 → HUSB238), one data-only (D+/D- → MPU USB OTG). Simpler routing than combining; trivially distinguishable by silkscreen. Both use the same TYPE-C-31-M-17 part for BoM consolidation.
- **Boot mode select**: 2-position DIP shown for SD vs USB serial-download recovery. i.MX 6ULL boot ROM supports many sources via BOOT_CFG fuses+pins; the DIP is a subset.
- **UART console**: CH340N + USB-A AND a 4-pin TTL header on the same MPU UART1 lines. Belt-and-suspenders for first bring-up.
