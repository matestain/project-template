# Template: Robotic Palletizer Cell — 1 Robot

**Template ID:** T-STD-015 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Single robot palletizing cell with perimeter guarding

---

## Scope

**Covers:**
Single industrial robot palletizing cell: infeed conveyor, product grouping, layer pattern execution via robot program, pallet conveyor, perimeter safety guarding with light curtains, pallet lot tracking.

**Does NOT cover:**
Dual robot (use T-STD-016), robot programming (see T-ROB-001/002), stretch wrapping (add as downstream machine).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1516F-3 PN (Safety) | GuardLogix 5580 L43-8ERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |
| Robot (palletizing) | KUKA KRC5 + KR QUANTEC PA | Yaskawa YRC1000 + MPL series |
| Safety scanner / light curtain | SICK microScan3 + S300 | SICK microScan3 + S300 |
| Pallet conveyor | SEW or Sinamics VFD driven | Rockwell Kinetix or VFD driven |

> Safety minimum: **PLd** | HMI level: **L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
- T-MOD-005 — Conveyor Zone
- T-MOD-006 — Product Counter / Encoder
- T-MOD-007 — Safety Door Monitor
- T-MOD-008 — E-Stop Chain
- T-MOD-009 — Light Curtain Interface
- T-MOD-010 — Robot Interface
- T-MOD-015 — Recipe Management
- T-MOD-016 — Shift Counter / Production Report
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-019 — OPC-UA Data Map
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-030 — Lot Management
- T-MOD-031 — Timer Extended
- T-MOD-032 — State Machine

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

> **Robot subfolder:** `robot/kuka/` and `robot/yaskawa/` — see T-ROB-XXX templates for robot program bases.

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
