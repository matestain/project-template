# Template: Valve Cluster — Medium

**Template ID:** T-STD-002 | **Version:** 0.1.0 | **Status:** 📋 Defined

> 16–40 valves, multi-routing up to 4 simultaneous circuits

---

## Scope

**Covers:**
16–40 on/off pneumatic valves with multi-routing logic supporting up to 4 simultaneous independent circuits. Recipe-based circuit selection.

**Does NOT cover:**
More than 40 valves (use T-STD-003), proportional valves, process control.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1214C DC/DC/DC or S7-1515-2 PN | CompactLogix 5380 L33ER or L36ER |
| HMI | KTP700 Basic 6" or KP700 Comfort 7" | PanelView 800 7" or PanelView Plus 7 10" |

> Safety minimum: **PLc** | HMI level: **L1–L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
- T-MOD-008 — E-Stop Chain
- T-MOD-018 — Alarm Buffer — Profile B
- T-MOD-019 — OPC-UA Data Map
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-031 — Timer Extended

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
