# Template: Pasteurizer

**Template ID:** T-STD-019 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Temperature-controlled pasteurization with holding tube and valve cluster

---

## Scope

**Covers:**
Continuous-flow pasteurizer: heating section, holding tube, cooling section, flow divert valve, temperature and flow PID loops, CIP preparation (valve cluster). HTST and LTLT recipes.

**Does NOT cover:**
CIP execution (use T-STD-022/023), product filling, homogenization.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1516F-3 PN (Safety) | GuardLogix 5380 L36ERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |
| Temperature sensors | PT100 / 4-20 mA | PT100 / 4-20 mA |
| Flow meter | Endress+Hauser Promag / Coriolis | Endress+Hauser Promag / Coriolis |

> Safety minimum: **PLc–PLd** | HMI level: **L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
- T-MOD-003 — Pneumatic Valve — Control
- T-MOD-004 — PID Control Loop
- T-MOD-007 — Safety Door Monitor
- T-MOD-008 — E-Stop Chain
- T-MOD-015 — Recipe Management
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-019 — OPC-UA Data Map
- T-MOD-020 — Process Sequence Block
- T-MOD-021 — Analog Input
- T-MOD-022 — Analog Output
- T-MOD-023 — Probe Management
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-026 — Scaling
- T-MOD-027 — Totalizer
- T-MOD-028 — Average
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
