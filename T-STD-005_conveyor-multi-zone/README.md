# Template: Conveyor System — Multi Zone

**Template ID:** T-STD-005 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Multiple drives, accumulation zones, zone-to-zone handoff

---

## Scope

**Covers:**
Multi-drive conveyor line with accumulation zones, photoeye arrays, zone-to-zone product handoff, and backpressure management.

**Does NOT cover:**
Sorting logic (use T-STD-006/007), palletizing infeed (use T-STD-010/015).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1515F-2 PN (Safety) | GuardLogix 5380 L36ERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 10" |

> Safety minimum: **PLd** | HMI level: **L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-005 — Conveyor Zone
- T-MOD-006 — Product Counter / Encoder
- T-MOD-008 — E-Stop Chain
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
