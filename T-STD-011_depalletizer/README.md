# Template: Depalletizer

**Template ID:** T-STD-011 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> Automatic pallet destacking, layer-by-layer feed to conveyor

---

## Scope

**Covers:**
Layer-by-layer depalletizer feeding products from full pallets onto an outfeed conveyor. Includes pallet lift, layer sweep, and empty pallet discharge.

**Does NOT cover:**
Robotic depalletizing, layer vision (add separately as project customization).

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1515F-2 PN (Safety) | GuardLogix 5380 L36ERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |

> Safety minimum: **PLd** | HMI level: **L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-002 — Pneumatic Valve — On/Off
- T-MOD-006 — Product Counter / Encoder
- T-MOD-008 — E-Stop Chain
- T-MOD-015 — Recipe Management
- T-MOD-016 — Shift Counter / Production Report
- T-MOD-017 — Alarm Buffer — Profile A
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
