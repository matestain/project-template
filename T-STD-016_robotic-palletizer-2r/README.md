# Template: Robotic Palletizer Cell — 2 Robots

**Template ID:** T-STD-016 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Dual robot palletizing, shared workspace management

---

## Scope

**Covers:**
Two-robot palletizing cell with shared workspace zone management, independent pallet stations, coordinated infeed splitting, and dual safety zone monitoring.

**Does NOT cover:**
Three or more robots, robot programming (see T-ROB-001/002).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1517F-3 PN (Safety) | ControlLogix 5580 GuardLogix L86ES |
| HMI | KP900 Comfort 9" + WinCC Unified PC (L3) | PanelView Plus 7 12" + FT View SE (L3) |
| Robot × 2 (palletizing) | KUKA KRC5 + KR QUANTEC PA × 2 | Yaskawa YRC1000 + MPL × 2 |
| Safety scanner | SICK microScan3 (per zone) | SICK microScan3 (per zone) |

> Safety minimum: **PLd** | HMI level: **L2–L3**

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
