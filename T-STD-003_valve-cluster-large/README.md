# Template: Valve Cluster — Large

**Template ID:** T-STD-003 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> 41+ valves, complex multi-routing, simultaneous circuit management

---

## Scope

**Covers:**
41+ on/off pneumatic valves with complex multi-routing and simultaneous circuit management. Intended for large CIP matrices and process distribution headers.

**Does NOT cover:**
Process-specific control loops (use T-STD-019+), proportional valves.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1515-2 PN or S7-1515F-2 PN (Safety) | CompactLogix 5380 L36ER or GuardLogix 5380 L3xERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |

> Safety minimum: **PLc–PLd** | HMI level: **L2**

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
