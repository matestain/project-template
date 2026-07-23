# Template: Valve Cluster — Small

**Template ID:** T-STD-001 | **Version:** 0.1.0 | **Status:** 📋 Defined

> < 15 valves, single circuit, no multi-routing

---

## Scope

**Covers:**
Up to 15 on/off and/or open-loop modulating pneumatic valves on a single circuit. Includes valve sequencing, position feedback, timeout faults, E-stop chain, and OPC-UA data map. Modulating valves are 3-position routing with direct AO output — no PID.

**Does NOT cover:**
Multi-circuit routing, more than 15 valves (use T-STD-002/003), hydraulic proportional valves with closed-loop position control.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1214C DC/DC/DC | CompactLogix 5380 L16ER |
| HMI | KTP700 Basic 6" | PanelView 800 7" |

> Safety minimum: **PLc** | HMI level: **L1**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
- T-MOD-003 — Pneumatic Valve — Control
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
