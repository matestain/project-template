# Template: Washer

**Template ID:** T-STD-024 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Bottle, tray, or container washing — spray zones, temperature, conveyor speed

---

## Scope

**Covers:**
Tunnel washer for bottles, trays, or reusable containers: spray zone control, wash water temperature monitoring, conveyor speed, detergent dosing enable.

**Does NOT cover:**
CIP (use T-STD-022), air drying / blow-off (add as project customization).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1515-2 PN | CompactLogix 5380 L36ER |
| HMI | KP700 Comfort 7" | PanelView Plus 7 10" |
| Temperature sensors | PT100 / 4-20 mA | PT100 / 4-20 mA |

> Safety minimum: **PLc** | HMI level: **L1–L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
- T-MOD-008 — E-Stop Chain
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-019 — OPC-UA Data Map
- T-MOD-020 — Process Sequence Block
- T-MOD-021 — Analog Input
- T-MOD-023 — Probe Management
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-026 — Scaling
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
