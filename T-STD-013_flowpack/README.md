# Template: Flowpack

**Template ID:** T-STD-013 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Horizontal form-fill-seal, continuous motion, film and product sync

---

## Scope

**Covers:**
Horizontal HFFS (flowpack) machine: film unwind, forming box, product infeed sync, longitudinal and transverse sealing, continuous-motion servo coordination.

**Does NOT cover:**
Vertical FFS (VFFS), thermoforming, robotic infeed.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1516F-3 PN (Safety) | GuardLogix 5380 L36ERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |
| Servo drives (film, jaw, infeed) | Sinamics S120 multi-axis | Kinetix 5700 |

> Safety minimum: **PLd** | HMI level: **L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
- T-MOD-006 — Product Counter / Encoder
- T-MOD-007 — Safety Door Monitor
- T-MOD-008 — E-Stop Chain
- T-MOD-015 — Recipe Management
- T-MOD-016 — Shift Counter / Production Report
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-019 — OPC-UA Data Map
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-031 — Timer Extended
- T-MOD-032 — State Machine
- T-MOD-037 — Servo / Motion Axis — Basic

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
