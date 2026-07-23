# Template: Robotic Box Former

**Template ID:** T-STD-018 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Robot-assisted box erecting and product loading

---

## Scope

**Covers:**
Robot-assisted carton erection from flat blanks and product loading. Includes blank magazine, robot forming sequence, hot-melt glue control, and loaded box discharge.

**Does NOT cover:**
Palletizing (use T-STD-015), robot programming (see T-ROB-005/006).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1515F-2 PN (Safety) | GuardLogix 5380 L3xERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 10" |
| Robot (box forming) | KUKA KRC5 + KR AGILUS | Yaskawa YRC1000 + MPK |
| Glue system | Nordson or Robatech | Nordson or Robatech |

> Safety minimum: **PLd** | HMI level: **L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
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
