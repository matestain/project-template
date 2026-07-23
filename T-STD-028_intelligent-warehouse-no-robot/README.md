# Template: Intelligent Warehouse — No Robot

**Template ID:** T-STD-028 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Automated storage and retrieval, conveyor-based, barcode/RFID tracking

---

## Scope

**Covers:**
Automated warehouse I/O conveyor system with barcode/RFID product tracking, storage location assignment, LGV fleet interface, SCADA-level visibility. No robotic pick stations.

**Does NOT cover:**
Robotic pick (use T-STD-029), WMS (3rd-party), ERP integration (project-specific).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1517F-3 PN (Safety) | ControlLogix 5580 GuardLogix L86ES |
| HMI | WinCC Unified PC (L3) | FT View SE Server |
| Barcode / RFID readers | Datalogic or SICK | Datalogic or SICK |
| ASRS crane / shuttle (optional) | 3rd-party EtherNet/IP | 3rd-party EtherNet/IP |
| LGV fleet (optional) | Elettric80 / Savant | Elettric80 / Savant |

> Safety minimum: **PLd** | HMI level: **L3**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-005 — Conveyor Zone
- T-MOD-007 — Safety Door Monitor
- T-MOD-008 — E-Stop Chain
- T-MOD-009 — Light Curtain Interface
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

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
