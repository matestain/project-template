# Template: Packer / Wrapper

**Template ID:** T-STD-012 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Product grouping and wrapping (shrink, stretch, flow)

---

## Scope

**Covers:**
Product grouping and film wrapping machine — shrink tunnel, stretch hood, or flow-wrap. Servo-driven film feed, recipe-based pack patterns, guarded access doors.

**Does NOT cover:**
Form-fill-seal (use T-STD-013), robotic packing (use T-STD-017).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1516F-3 PN (Safety) | GuardLogix 5380 L36ERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |
| Servo drive (grouping / film feed) | Sinamics S210 | Kinetix 5500 |

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
