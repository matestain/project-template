# Template: Furnace Line

**Template ID:** T-STD-025 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Post-oven line — tray dispensing, tray cleaning, cooling conveyor, product transfer

---

## Scope

**Covers:**
Complete post-oven tray handling line: tray dispensing from stack, tray cleaning station, oven-exit transfer, multi-zone cooling conveyor, product transfer to packing.

**Does NOT cover:**
Oven control (client equipment), product depositing, palletizing of finished goods.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1516F-3 PN (Safety) | GuardLogix 5380 L36ERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |
| Temperature sensors (cooling zone) | PT100 / 4-20 mA | PT100 / 4-20 mA |

> Safety minimum: **PLc–PLd** | HMI level: **L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
- T-MOD-005 — Conveyor Zone
- T-MOD-006 — Product Counter / Encoder
- T-MOD-008 — E-Stop Chain
- T-MOD-016 — Shift Counter / Production Report
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-019 — OPC-UA Data Map
- T-MOD-021 — Analog Input
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-026 — Scaling
- T-MOD-029 — FIFO / LIFO Buffer
- T-MOD-031 — Timer Extended
- T-MOD-032 — State Machine
- T-MOD-038 — Tray / Dispenser Sequence
- T-MOD-039 — Cooling Zone — Process

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
