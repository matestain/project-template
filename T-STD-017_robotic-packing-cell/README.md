# Template: Robotic Packing Cell

**Template ID:** T-STD-017 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Pick-and-place packing, vision optional, variable product

---

## Scope

**Covers:**
Robotic pick-and-place packing cell: product infeed, robot pick with tool changer, case/tray loading, vision interface (optional), outfeed to sealing or palletizing.

**Does NOT cover:**
Box forming (use T-STD-018), palletizing (use T-STD-015/016), robot programming (see T-ROB-003/004).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1516F-3 PN (Safety) | GuardLogix 5580 L43-8ERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |
| Robot (pick & place) | KUKA KRC5 + KR AGILUS or SCARA | Yaskawa YRC1000 + MPP/MPK |
| Vision system (optional) | Cognex In-Sight / 3rd-party | Cognex In-Sight / 3rd-party |
| Light curtain / safety scanner | SICK S300 / microScan3 | SICK S300 / microScan3 |

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
