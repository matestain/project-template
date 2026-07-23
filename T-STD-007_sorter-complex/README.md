# Template: Sorter — Complex

**Template ID:** T-STD-007 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Dynamic destination, vision/weight classification, high-speed

---

## Scope

**Covers:**
High-speed sorter with dynamic destination assignment driven by barcode or weight classification. Intended for logistics, postal, and airport applications.

**Does NOT cover:**
Fixed-pattern sorting (use T-STD-006), robotic sorting.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1516F-3 PN (Safety) | GuardLogix 5580 L43-8ERS |
| HMI | KP900 Comfort 9" + WinCC Unified PC (L3) | PanelView Plus 7 10" + FT View SE (L3) |
| Barcode / weight scanner | 3rd-party serial/EtherNet | 3rd-party serial/EtherNet |

> Safety minimum: **PLd** | HMI level: **L2–L3**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-005 — Conveyor Zone
- T-MOD-006 — Product Counter / Encoder
- T-MOD-008 — E-Stop Chain
- T-MOD-015 — Recipe Management
- T-MOD-016 — Shift Counter / Production Report
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-019 — OPC-UA Data Map
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-029 — FIFO / LIFO Buffer
- T-MOD-031 — Timer Extended
- T-MOD-032 — State Machine
- T-MOD-034 — Barcode / RFID Reader Interface
- T-MOD-035 — Weight / Scale Interface

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
