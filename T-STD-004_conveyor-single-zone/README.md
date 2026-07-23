# Template: Conveyor System — Single Zone

**Template ID:** T-STD-004 | **Version:** 0.1.0 | **Status:** 📋 Defined

> Single drive, product transport, jam detection

---

## Scope

**Covers:**
Single-drive belt or roller conveyor with photoeye detection, jam logic, product counting, and shift production tracking.

**Does NOT cover:**
Multiple zones or accumulation (use T-STD-005), sorter logic, zone-to-zone handoff.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1214C DC/DC/DC | CompactLogix 5380 L18ER |
| HMI | KTP700 Basic 6" | PanelView 800 7" |

> Safety minimum: **PLc** | HMI level: **L1**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-005 — Conveyor Zone
- T-MOD-006 — Product Counter / Encoder
- T-MOD-008 — E-Stop Chain
- T-MOD-016 — Shift Counter / Production Report
- T-MOD-018 — Alarm Buffer — Profile B
- T-MOD-019 — OPC-UA Data Map
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
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
