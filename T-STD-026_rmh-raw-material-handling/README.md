# Template: RMH — Raw Material Handling

**Template ID:** T-STD-026 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Bulk material receiving, weighing, conveying to process

---

## Scope

**Covers:**
Bulk raw material receiving station: truck or rail unloading, silo level monitoring, weigh belt / batch scale, conveying to process silos or day bins, lot traceability.

**Does NOT cover:**
Process-side consumption control, ingredient dosing.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1515-2 PN | CompactLogix 5380 L36ER |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |
| Weigh belt / scale | Schenck or Siwarex | Hardy Process Solutions |
| Level sensor (silo) | 4-20 mA radar / ultrasonic | 4-20 mA radar / ultrasonic |

> Safety minimum: **PLc** | HMI level: **L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
- T-MOD-008 — E-Stop Chain
- T-MOD-016 — Shift Counter / Production Report
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-019 — OPC-UA Data Map
- T-MOD-021 — Analog Input
- T-MOD-023 — Probe Management
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-026 — Scaling
- T-MOD-027 — Totalizer
- T-MOD-030 — Lot Management
- T-MOD-031 — Timer Extended
- T-MOD-032 — State Machine
- T-MOD-035 — Weight / Scale Interface

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
