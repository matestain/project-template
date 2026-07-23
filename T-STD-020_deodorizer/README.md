# Template: Deodorizer

**Template ID:** T-STD-020 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Vacuum/steam deodorization process, temperature and pressure control

---

## Scope

**Covers:**
Vacuum/steam deodorization column: vacuum pump control, steam injection PID, temperature and pressure multi-loop management, stripping column phases, recipe-driven process program.

**Does NOT cover:**
Refining or bleaching upstream stages, CIP (use T-STD-022/023).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1517F-3 PN (Safety) | ControlLogix 5580 GuardLogix L83ES |
| HMI | KP900 Comfort 9" + WinCC Unified PC (L3) | PanelView Plus 7 12" + FT View SE (L3) |
| Vacuum pump | Nash / Busch — 4-20 mA control | Nash / Busch — 4-20 mA control |
| Pressure / temp transmitters | Endress+Hauser 4-20 mA | Endress+Hauser 4-20 mA |

> Safety minimum: **PLd** | HMI level: **L2–L3**

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
