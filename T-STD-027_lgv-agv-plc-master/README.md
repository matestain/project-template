# Template: LGV / AGV — PLC Master

**Template ID:** T-STD-027 | **Version:** 0.1.0 | **Status:** ⬜ Planned

> PLC as fleet master or manual station intermediary for guided vehicles

---

## Scope

**Covers:**
PLC master controller interfacing to a 3rd-party LGV fleet manager: call/release handshake, task assignment over EtherNet/IP or OPC-UA, safety zone isolation, manual override stations.

**Does NOT cover:**
LGV onboard programming (3rd-party responsibility), warehouse management system (WMS) integration.

---

## Hardware Reference BOM

| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-1516F-3 PN (Safety) | GuardLogix 5580 L43-8ERS |
| HMI | KP900 Comfort 9" | PanelView Plus 7 12" |
| LGV fleet manager | 3rd-party (Elettric80, Savant, etc.) | 3rd-party |
| Floor safety scanners | SICK microScan3 (per aisle) | SICK microScan3 (per aisle) |

> Safety minimum: **PLd** | HMI level: **L2**

---

## Common Modules Used

- T-MOD-001 — Motor Control
- T-MOD-008 — E-Stop Chain
- T-MOD-016 — Shift Counter / Production Report
- T-MOD-017 — Alarm Buffer — Profile A
- T-MOD-019 — OPC-UA Data Map
- T-MOD-024 — Digital Input
- T-MOD-025 — Digital Output
- T-MOD-031 — Timer Extended
- T-MOD-032 — State Machine
- T-MOD-036 — LGV / AGV Interface

> All modules live in `../../common/`. Do not duplicate them inside this folder.
> Symlinks to used modules belong in `common-modules/`.

---

## Version History

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-04 | Folder structure created |

## Known Limitations

_None yet — to be populated during design phase._
