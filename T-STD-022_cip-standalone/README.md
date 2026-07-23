# Template: CIP — Standalone

**Template ID:** T-STD-022 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Clean-in-place for a single equipment unit (tank, mixer, filler)

---

## Scope

**Covers:**
CIP unit for a single equipment item: pre-rinse, caustic, intermediate rinse, acid, final rinse phases. Temperature and concentration control, phase timers, recipe selection.

**Does NOT cover:**
Multi-circuit centralized CIP (use T-STD-023), process product control (use T-STD-019+).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1515-2 PN | CompactLogix 5380 L36ER |
| HMI | KP700 Comfort 7" | PanelView Plus 7 10" |
| Conductivity / pH sensor | 4-20 mA Endress+Hauser | 4-20 mA Endress+Hauser |
| Flow meter | Endress+Hauser Promag | Endress+Hauser Promag |

> Safety minimum: **PLc** | HMI level: **L1–L2**

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
- T-MOD-033 — CIP Sequence

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
