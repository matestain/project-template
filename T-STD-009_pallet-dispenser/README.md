# Template: Pallet Dispenser

**Template ID:** T-STD-009 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Automatic pallet feeding from stack to conveyor or robot cell

---

## Scope

**Covers:**
Automatic pallet destacker dispensing single pallets from a gravity or mechanical stack to an outfeed conveyor or robotic cell infeed.

**Does NOT cover:**
Full palletizing or depalletizing sequences (use T-STD-010/011).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1212C DC/DC/DC | CompactLogix 5380 L16ER |
| HMI | KTP700 Basic 6" | PanelView 800 7" |

> Safety minimum: **PLc** | HMI level: **L1**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-008 — E-Stop Chain
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-018 — Alarm Buffer — Profile B
- T-MOD-019 — OPC-UA Data Map
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
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
