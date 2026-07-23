# Template: Intelligent Warehouse — With Robots

**Template ID:** T-STD-029 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> AS/RS with robotic pick stations, multi-zone safety management

---

## Scope

**Covers:**
Automated warehouse with robotic pick stations: I/O conveyor system, barcode/RFID tracking, multi-zone safety management per robot cell, LGV interface, SCADA-level visibility.

**Does NOT cover:**
Robot programming (see T-ROB-007/008 for cell init), WMS / ERP integration (project-specific).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1517F-3 PN (Safety) | ControlLogix 5580 GuardLogix L86ES |
| HMI | WinCC Unified PC (L3) | FT View SE Server |
| Robot(s) (pick stations) | KUKA KRC5 / Yaskawa YRC1000 | KUKA KRC5 / Yaskawa YRC1000 |
| Barcode / RFID readers | Datalogic or SICK | Datalogic or SICK |
| Safety scanners (per zone) | SICK microScan3 | SICK microScan3 |
| LGV fleet (optional) | Elettric80 / Savant | Elettric80 / Savant |

> Safety minimum: **PLd** | HMI level: **L3**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-005 — Conveyor Zone
- T-MOD-007 — Safety Door Monitor
- T-MOD-008 — E-Stop Chain
- T-MOD-009 — Light Curtain Interface
- T-MOD-010 — Robot Interface
- T-MOD-016 — Shift Counter / Production Report
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-019 — OPC-UA Data Map
- T-MOD-021 — Analog Input
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-026 — Scaling
- T-MOD-030 — Lot Management
- T-MOD-031 — Timer Extended
- T-MOD-032 — State Machine
- T-MOD-034 — Barcode / RFID Reader Interface
- T-MOD-036 — LGV / AGV Interface

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
