# Template: Piler

**Template ID:** T-STD-008 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Generic product stacker — flat or shaped products, configurable stack count and pattern

---

## Scope

**Covers:**
Servo-driven product stacker for flat or shaped products. Configurable stack count, offset patterns, and infeed conveyor zone.

**Does NOT cover:**
Palletizing (use T-STD-010/015), robotic stacking.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1515F-2 PN (Safety) | GuardLogix 5380 L3xERS |
| HMI | KP700 Comfort 7" | PanelView Plus 7 10" |
| Servo drive (stacker axis) | Sinamics S120 or S210 | Kinetix 5500 |

> Safety minimum: **PLc–PLd** | HMI level: **L1–L2**

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
- T-MOD-031 — Timer Extended
- T-MOD-032 — State Machine
- T-MOD-037 — Servo / Motion Axis — Basic

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
