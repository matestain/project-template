# Template: Sorter — Simple

**Template ID:** T-STD-006 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> 1 input, multiple fixed outputs, recipe-based destination selection

---

## Scope

**Covers:**
Single-inlet conveyor sorter with multiple fixed output lanes. Recipe-based destination selection, divert gate control, product tracking.

**Does NOT cover:**
Vision or weight classification (use T-STD-007), dynamic destination assignment.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1515-2 PN | CompactLogix 5380 L36ER |
| HMI | KP700 Comfort 7" | PanelView Plus 7 10" |

> Safety minimum: **PLc** | HMI level: **L1–L2**

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

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
