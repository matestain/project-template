# Project Template Master List

> Version: 0.3.0 | Status: Active
> Last updated: 2024-01

---

## How to Use This Document

Master registry of all MATESTAIN project templates. Templates are built incrementally — this list tracks what exists, what is in progress, and what is planned.

**Status legend:**

| Status | Meaning |
|---|---|
| `✅ Complete` | Fully built, validated on at least one real project |
| `🔨 In Progress` | Under active development |
| `📋 Defined` | Structure defined, not yet built |
| `⬜ Planned` | In scope, not yet started |

**Template ID convention:** `T-[Category]-[NNN]`

---

## 1. Standalone Machine Templates

Every machine is an independent deliverable. Composite lines (conveyor + packer + palletizer) are always separate projects with defined interfaces between them.

### 1.1 Material Handling

| ID | Machine Type | Description | HMI Level | Safety Min. | Status |
|---|---|---|---|---|---|
| T-STD-001 | Valve Cluster — Small | < 15 valves, single circuit, no multi-routing | L1 | PLc | 📋 Defined |
| T-STD-002 | Valve Cluster — Medium | 16–40 valves, multi-routing up to 4 simultaneous circuits | L1–L2 | PLc | 📋 Defined |
| T-STD-003 | Valve Cluster — Large | 41+ valves, complex multi-routing, simultaneous circuit management | L2 | PLc–PLd | ⬜ Planned |
| T-STD-004 | Conveyor System — Single Zone | Single drive, product transport, jam detection | L1 | PLc | 📋 Defined |
| T-STD-005 | Conveyor System — Multi Zone | Multiple drives, accumulation zones, zone-to-zone handoff | L2 | PLd | ⬜ Planned |
| T-STD-006 | Sorter — Simple | 1 input, multiple fixed outputs, recipe-based destination selection | L1–L2 | PLc | ⬜ Planned |
| T-STD-007 | Sorter — Complex | Dynamic destination, vision/weight classification, high-speed (logistics, airports, postal) | L2–L3 | PLd | ⬜ Planned |
| T-STD-008 | Piler | Generic product stacker — flat or shaped products, configurable stack count and pattern | L1–L2 | PLc–PLd | ⬜ Planned |
| T-STD-009 | Pallet Dispenser | Automatic pallet feeding from stack to conveyor or robot cell | L1 | PLc | ⬜ Planned |

### 1.2 Packaging

| ID | Machine Type | Description | HMI Level | Safety Min. | Status |
|---|---|---|---|---|---|
| T-STD-010 | Palletizer — Layer | Layer-by-layer palletizing, fixed pattern, no robot | L2 | PLd | ⬜ Planned |
| T-STD-011 | Depalletizer | Automatic pallet destacking, layer-by-layer feed to conveyor | L2 | PLd | ⬜ Planned |
| T-STD-012 | Packer / Wrapper | Product grouping and wrapping (shrink, stretch, flow) | L2 | PLd | ⬜ Planned |
| T-STD-013 | Flowpack | Horizontal form-fill-seal, continuous motion, film and product sync | L2 | PLd | ⬜ Planned |
| T-STD-014 | Box Former / Assembler | Flat blank to erected box, glue or tape sealing | L2 | PLd | ⬜ Planned |
| T-STD-015 | Robotic Palletizer Cell — 1 Robot | Single robot palletizing cell with perimeter guarding | L2 | PLd | ⬜ Planned |
| T-STD-016 | Robotic Palletizer Cell — 2 Robots | Dual robot palletizing, shared workspace management | L2–L3 | PLd | ⬜ Planned |
| T-STD-017 | Robotic Packing Cell | Pick-and-place packing, vision optional, variable product | L2 | PLd | ⬜ Planned |
| T-STD-018 | Robotic Box Former | Robot-assisted box erecting and product loading | L2 | PLd | ⬜ Planned |

### 1.3 Process

| ID | Machine Type | Description | HMI Level | Safety Min. | Status |
|---|---|---|---|---|---|
| T-STD-019 | Pasteurizer | Temperature-controlled pasteurization with holding tube and valve cluster | L2 | PLc–PLd | ⬜ Planned |
| T-STD-020 | Deodorizer | Vacuum/steam deodorization process, temperature and pressure control | L2–L3 | PLd | ⬜ Planned |
| T-STD-021 | Mixer | Product mixing — speed, time, temperature recipe control | L1–L2 | PLc | ⬜ Planned |
| T-STD-022 | CIP — Standalone | Clean-in-place for a single equipment unit (tank, mixer, filler) | L1–L2 | PLc | ⬜ Planned |
| T-STD-023 | CIP — Centralized | Multi-circuit CIP managing cleaning of multiple plant equipment simultaneously | L2–L3 | PLc–PLd | ⬜ Planned |
| T-STD-024 | Washer | Bottle, tray, or container washing — spray zones, temperature, conveyor speed | L1–L2 | PLc | ⬜ Planned |
| T-STD-025 | Furnace Line | Post-oven line — tray dispensing, tray cleaning, cooling conveyor, product transfer | L2 | PLc–PLd | ⬜ Planned |

### 1.4 Intralogistics

| ID | Machine Type | Description | HMI Level | Safety Min. | Status |
|---|---|---|---|---|---|
| T-STD-026 | RMH — Raw Material Handling | Bulk material receiving, weighing, conveying to process | L2 | PLc | ⬜ Planned |
| T-STD-027 | LGV / AGV — PLC Master | PLC as fleet master or manual station intermediary for guided vehicles | L2 | PLd | ⬜ Planned |
| T-STD-028 | Intelligent Warehouse — No Robot | Automated storage and retrieval, conveyor-based, barcode/RFID tracking | L3 | PLd | ⬜ Planned |
| T-STD-029 | Intelligent Warehouse — With Robots | AS/RS with robotic pick stations, multi-zone safety management | L3 | PLd | ⬜ Planned |

---

## 2. Functional Module Templates

Built once in `project-template/common/`, referenced by multiple standalone templates. Never duplicated.

### Design decisions pending

| Module | Decision | When resolved |
|---|---|---|
| Safety modules (T-MOD-007, T-MOD-009) | Architecture depth defined per project in design phase — risk of over-engineering is real | First project requiring PLd robotic cell |
| Robot Interface (T-MOD-010) | Goal: single unified UDT for KUKA + Yaskawa on TIA + RS5000, with platform variant only in comms layer | Design phase before T-STD-015 |
| Probe Management (T-MOD-023) | Unified (one block, `ProbeType` parameter) vs split (one per signal type) — depends on diagnostic logic divergence | Design phase before T-STD-019 |
| VFD comms — native platforms | TIA + Sinamics and RS5000 + PowerFlex use native function blocks — no auxiliary module needed | N/A — no module built |

### Module List

| ID | Module | Description | Notes |
|---|---|---|---|
| T-MOD-001 | Motor Control | Unified DOL + VFD: run/stop, speed SP/FB, fault, overload, thermal. `MotorType` param selects DOL/VFD | DOL and VFD unified |
| T-MOD-002 | Pneumatic Valve — On/Off | Single/dual solenoid, position feedback (open/closed/moving), timeout fault | |
| T-MOD-003 | Pneumatic Valve — Control | Proportional valve, position SP/FB, control loop, limits | |
| T-MOD-004 | PID Control Loop | Generic PID: auto/manual, SP ramp, output limits, integral windup protection | |
| T-MOD-005 | Conveyor Zone | Zone logic: photoeye, accumulation, release, jam detection, zone-to-zone handoff | Standalone — instanced N times |
| T-MOD-006 | Product Counter / Encoder | Pulse counter, encoder input, rate (units/min), batch counter, reset | |
| T-MOD-007 | Safety Door Monitor | Cat. 3 dual-channel door interlock, cross-monitoring, feedback | Architecture: design phase |
| T-MOD-008 | E-Stop Chain | E-stop chain management, zone isolation, reset logic, state feedback | Universal |
| T-MOD-009 | Light Curtain Interface | Dual-channel light curtain, muting logic (2 or 4 sensor), override with key | Architecture: design phase |
| T-MOD-010 | Robot Interface | Unified PLC ↔ Robot handshake UDT — program select, cycle, fault mapping, safety interface | Unified UDT: design phase |
| T-MOD-011 | VFD Comms — Danfoss | Danfoss FC/VLT parameter word → T-MOD-001 UDT | Cross-vendor only |
| T-MOD-012 | VFD Comms — Schneider ATV | Schneider Altivar parameter word → T-MOD-001 UDT | Cross-vendor only |
| T-MOD-013 | VFD Comms — SEW Eurodrive | SEW MOVI parameter word → T-MOD-001 UDT | Cross-vendor only |
| T-MOD-014 | VFD Comms — Lenze | Lenze i500/8400 parameter word → T-MOD-001 UDT | Cross-vendor only |
| T-MOD-015 | Recipe Management | Unified for material handling + process. `RecipeType` param adapts UDT size. Select, load/save, version, access control | Material handling + process unified |
| T-MOD-016 | Shift Counter / Production Report | Shift start/end, good count, reject count, downtime accumulation, OEE inputs | |
| T-MOD-017 | Alarm Handler | Single alarm trigger, Severity, Category, Device filters, Timestamps, optional history record if enabled for systems without HMI | Low resource draw, highly expandable with T-MOD-018 Alarm Manager |
| T-MOD-018 | Alarm Manager | Wrap for T-MOD-17 Alarm Handler | Compatible with all systems |
| T-MOD-019 | OPC-UA Data Map | Tag-to-node mapping, server config, Plantwise entry point | Universal |
| T-MOD-020 | Process Sequence Block | Generic SKID container: phases (Init/Run/Hold/Abort/Clean), state machine, interlock inputs, alarm outputs. One instance per SKID | Modular process templates |
| T-MOD-021 | Analog Input | Raw AI → engineering units, wire-break, out-of-range alarm, filter. Writes standard Analog UDT | Foundation for T-MOD-023 |
| T-MOD-022 | Analog Output | Engineering units → raw AO, output clamp, feedback verification. Writes standard Analog UDT | |
| T-MOD-023 | Probe Management | Wrapper over T-MOD-021: parametrization, error capture, overflow, calibration offset. Covers 4-20mA, 0-10V, ±10V, PT100 | Unified vs split: TBD in design |
| T-MOD-024 | Digital Input | DI with configurable debounce, inversion, pulse detection. Writes standard Digital UDT | Universal |
| T-MOD-025 | Digital Output | DO with output feedback verification, force/override. Writes standard Digital UDT | Universal |
| T-MOD-026 | Scaling | Linear scaling (X→Y), dead-band, clamp. Standalone utility | Useful standalone for non-instrument signals |
| T-MOD-027 | Totalizer | Accumulator with reset (shift/lot/manual), units, overflow protection, trend output | |
| T-MOD-028 | Average | Simple and moving average (configurable window), outlier rejection option | |
| T-MOD-029 | FIFO / LIFO Buffer | Generic typed buffer, configurable depth, push/pop/peek, overflow alarm | Product tracking + lot sequencing |
| T-MOD-030 | Lot Management | Lot ID, timestamp, quantity tracking, FIFO integration for pallet/product traceability | Combines with T-MOD-029 |
| T-MOD-031 | Timer Extended | Wrapper over native timer: retrigger, elapsed time logging, timeout with integrated alarm output | Universal |
| T-MOD-032 | State Machine | Generic state machine: state enum, transition logic, dwell timer, state log (last N transitions) | Standardizes sequence debugging |
| T-MOD-033 | CIP Sequence | Standard CIP phases: pre-rinse, caustic, rinse, acid, final rinse. Phase times and concentrations from recipe | |
| T-MOD-034 | Barcode / RFID Reader Interface | Serial/EtherNet reader, product ID decode, routing decision output | |
| T-MOD-035 | Weight / Scale Interface | Analog or digital scale, tare, gross/net, over/under alarm, lot weight accumulation | |
| T-MOD-036 | LGV / AGV Interface | PLC ↔ LGV fleet manager handshake, call/release, task assignment, fault | |
| T-MOD-037 | Servo / Motion Axis — Basic | Single axis: home, jog, position table, velocity profile, fault | |
| T-MOD-038 | Tray / Dispenser Sequence | Tray pick from stack, cleaning station, placement, count | |
| T-MOD-039 | Cooling Zone — Process | Cooling conveyor, zone temperature monitoring, fan/damper control | |

---

## 3. Robot Program Templates

| ID | Program Type | Platform | Used in |
|---|---|---|---|
| T-ROB-001 | Palletizer — Layer pattern | KUKA KRC4/KRC5 | T-STD-015, T-STD-016 |
| T-ROB-002 | Palletizer — Layer pattern | Yaskawa YRC1000 | T-STD-015, T-STD-016 |
| T-ROB-003 | Pick and place — Packing | KUKA KRC4/KRC5 | T-STD-017 |
| T-ROB-004 | Pick and place — Packing | Yaskawa YRC1000 | T-STD-017 |
| T-ROB-005 | Box forming — Robotic | KUKA KRC4/KRC5 | T-STD-018 |
| T-ROB-006 | Box forming — Robotic | Yaskawa YRC1000 | T-STD-018 |
| T-ROB-007 | Cell initialization — Generic | KUKA KRC4/KRC5 | All KUKA T-STD |
| T-ROB-008 | Cell initialization — Generic | Yaskawa YRC1000 | All Yaskawa T-STD |

---

## 4. HMI / SCADA Templates

| ID | Template | Platform | Level | Used in |
|---|---|---|---|---|
| T-HMI-001 | Faceplate library | WinCC Comfort | L1–L2 | All Siemens T-STD |
| T-HMI-002 | Faceplate library | WinCC Unified | L3 | Siemens L3 T-STD |
| T-HMI-003 | Faceplate library | FT View ME | L1–L2 | All Rockwell T-STD |
| T-HMI-004 | Faceplate library | FT View SE | L3 | Rockwell L3 T-STD |
| T-HMI-005 | Navigation + alarm screen | WinCC Comfort | L1–L2 | All Siemens T-STD |
| T-HMI-006 | Navigation + alarm screen | WinCC Unified | L3 | Siemens L3 |
| T-HMI-007 | Navigation + alarm screen | FT View ME | L1–L2 | All Rockwell T-STD |
| T-HMI-008 | Overview — Valve cluster | WinCC Comfort | L1–L2 | T-STD-001 through T-STD-003 |
| T-HMI-009 | Overview — Conveyor / Sorter | WinCC Comfort | L1–L2 | T-STD-004 through T-STD-007 |
| T-HMI-010 | Overview — Palletizer / Packer | WinCC Comfort | L2 | T-STD-010 through T-STD-014 |
| T-HMI-011 | Overview — Robotic cell | WinCC Comfort | L2 | T-STD-015 through T-STD-018 |
| T-HMI-012 | Overview — Process | WinCC Comfort | L2 | T-STD-019 through T-STD-024 |
| T-HMI-013 | Overview — Furnace line | WinCC Comfort | L2 | T-STD-025 |
| T-HMI-014 | Overview — Warehouse | WinCC Unified | L3 | T-STD-028, T-STD-029 |
| T-HMI-015 | Recipe screen | WinCC Comfort | L2 | All L2 Siemens T-STD |
| T-HMI-016 | Trend screen | WinCC Comfort | L2 | All L2 Siemens T-STD |
| T-HMI-017 | CIP sequence screen | WinCC Comfort | L2 | T-STD-022, T-STD-023 |
| T-HMI-018 | OEE / production dashboard | WinCC Unified | L3 | All L3 T-STD |

---

## 5. Document Templates

| ID | Document | Format | Used in | Status |
|---|---|---|---|---|
| T-DOC-001 | Alarm Register | `.xlsx` | All T-STD | ✅ Complete |
| T-DOC-002 | I/O List | `.xlsx` | All T-STD | ⬜ Planned |
| T-DOC-003 | Network Topology | `.xlsx` | All T-STD | ⬜ Planned |
| T-DOC-004 | Tag Cross-Reference | `.xlsx` | T-STD with robots or SCADA | ⬜ Planned |
| T-DOC-005 | Commissioning Log | `.xlsx` | All T-STD | ⬜ Planned |
| T-DOC-006 | Risk Assessment — Valve Cluster | `.xlsx` | T-STD-001 through 003 | ⬜ Planned |
| T-DOC-007 | Risk Assessment — Conveyor / Sorter | `.xlsx` | T-STD-004 through 007 | ⬜ Planned |
| T-DOC-008 | Risk Assessment — Palletizer / Packer | `.xlsx` | T-STD-010 through 014 | ⬜ Planned |
| T-DOC-009 | Risk Assessment — Robotic Cell | `.xlsx` | T-STD-015 through 018 | ⬜ Planned |
| T-DOC-010 | Risk Assessment — Process | `.xlsx` | T-STD-019 through 024 | ⬜ Planned |
| T-DOC-011 | Risk Assessment — Furnace / Intralogistics | `.xlsx` | T-STD-025 through 029 | ⬜ Planned |
| T-DOC-012 | Safety Validation Plan | `.xlsx` | All T-STD with safety functions | ⬜ Planned |
| T-DOC-013 | Recipe Sheet | `.xlsx` | T-STD with recipes | ⬜ Planned |
| T-DOC-014 | Cybersecurity Handover | `.md` | All T-STD | ⬜ Planned |
| T-DOC-015 | Delivery Checklist (pre-filled per type) | `.md` | All T-STD | ⬜ Planned |
| T-DOC-016 | MOC Template (pre-filled per type) | `.md` | All T-STD | ⬜ Planned |
| T-DOC-017 | Operator Manual + Troubleshooting | `.docx` | All T-STD | ⬜ Planned |
| T-DOC-018 | Training Presentation | `.pptx` | All T-STD | ⬜ Planned |
| T-DOC-019 | Competency Checklist | `.xlsx` | All T-STD | ⬜ Planned |
| T-DOC-020 | Operator Exam | `.docx` | All T-STD | ⬜ Planned |
| T-DOC-021 | Technical File Index (CE marking) | `.md` | CE projects only | ⬜ Planned |

**Notes on T-DOC-017 through T-DOC-020:**
- Templates in English. When a project is generated, files are translated to client language. Both versions delivered.
- T-DOC-017 has two layers: Layer 1 (operator) — symptom → immediate action → when to call maintenance, plain language, HMI-based. Layer 2 (technician/engineer) — Alarm ID → probable root cause → field checks → program block/tag reference. Covers new MATESTAIN service engineers on first site visit.
- T-DOC-021 is optional — required only for CE marking projects. Groups all safety documents into a formal Technical File package per Machinery Regulation 2023/1230.

---

## 6. Common Module Cross-Reference

`✓` = module used in template. `Count` drives build priority. VFD comms (T-MOD-011 through T-MOD-014) are on-demand — built when the first project with that drive brand appears.

| Module | 001 | 002 | 003 | 004 | 005 | 006 | 007 | 008 | 009 | 010 | 011 | 012 | 013 | 014 | 015 | 016 | 017 | 018 | 019 | 020 | 021 | 022 | 023 | 024 | 025 | 026 | 027 | 028 | 029 | **Count** |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| T-MOD-008 E-Stop | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **29** |
| T-MOD-019 OPC-UA | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **29** |
| T-MOD-024 DI | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **29** |
| T-MOD-025 DO | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **29** |
| T-MOD-001 Motor | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **29** |
| T-MOD-031 Timer | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **29** |
| T-MOD-032 State machine | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **26** |
| T-MOD-017 Alarm A | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **25** |
| T-MOD-002 Valve on/off | ✓ | ✓ | ✓ | | | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | | | | **20** |
| T-MOD-016 Shift ctr | | | | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | | ✓ | | | ✓ | ✓ | **17** |
| T-MOD-006 Counter | | | | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | | ✓ | | | | | **16** |
| T-MOD-015 Recipe | | | | | | ✓ | ✓ | ✓ | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | | **15** |
| T-MOD-005 Conv. zone | | | | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | | ✓ | ✓ | ✓ | | | | | | | | ✓ | | | ✓ | ✓ | **11** |
| T-MOD-021 AI | | | | | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | ✓ | **10** |
| T-MOD-026 Scaling | | | | | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | ✓ | **10** |
| T-MOD-007 Door mon. | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | | ✓ | ✓ | **12** |
| T-MOD-018 Alarm B | ✓ | ✓ | ✓ | ✓ | | | | | ✓ | | | | | | | | | | | | | | | | | | | | | **5** |
| T-MOD-023 Probe | | | | | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ | | | | **7** |
| T-MOD-020 Proc. seq. | | | | | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | **6** |
| T-MOD-029 FIFO/LIFO | | | | | ✓ | ✓ | ✓ | | | | | | | | | | | | | | | | | | ✓ | | | ✓ | ✓ | **6** |
| T-MOD-010 Robot iface | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | | | | | | | | | | | ✓ | **5** |
| T-MOD-004 PID | | | | | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | | **5** |
| T-MOD-027 Totalizer | | | | | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | | | ✓ | | | | **6** |
| T-MOD-028 Average | | | | | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | | **5** |
| T-MOD-009 Light curt. | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | ✓ | | | | | | | | | | ✓ | ✓ | **6** |
| T-MOD-030 Lot mgmt | | | | | | | | | | | | | | | ✓ | ✓ | | | | | | | | | | ✓ | | ✓ | ✓ | **5** |
| T-MOD-022 AO | | | | | | | | | | | | | | | | | | | ✓ | ✓ | | ✓ | ✓ | | | | | | | **4** |
| T-MOD-003 Valve ctrl | | | | | | | | | | | | | | | | | | | ✓ | ✓ | | ✓ | ✓ | | | | | | | **4** |
| T-MOD-033 CIP seq. | | | | | | | | | | | | | | | | | | | | | | ✓ | ✓ | | | | | | | **2** |
| T-MOD-034 Barcode/RFID | | | | | | | ✓ | | | | | | | | | | | | | | | | | | | | | ✓ | ✓ | **3** |
| T-MOD-035 Scale | | | | | | | ✓ | | | | | | | | | | | | | | ✓ | | | | | ✓ | | | | **3** |
| T-MOD-036 LGV iface | | | | | | | | | | | | | | | | | | | | | | | | | | | ✓ | ✓ | ✓ | **3** |
| T-MOD-037 Servo axis | | | | | | | | ✓ | | | | | ✓ | ✓ | | | | | | | | | | | | | | | | **3** |
| T-MOD-038 Tray disp. | | | | | | | | | | | | | | | | | | | | | | | | | ✓ | | | | | **1** |
| T-MOD-039 Cooling zone | | | | | | | | | | | | | | | | | | | | | | | | | ✓ | | | | | **1** |
| T-MOD-011–014 VFD comms | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | **on demand** |

---

## 7. Standard Template File Structure

```
project-template/
├── common/                        — Shared modules, built once
│   ├── T-MOD-001_motor/
│   ├── T-MOD-008_estop/
│   ├── T-MOD-019_opcua-map/
│   ├── T-MOD-024_digital-in/
│   ├── T-MOD-025_digital-out/
│   ├── T-MOD-031_timer-extended/
│   ├── T-MOD-032_state-machine/
│   └── ...
│
└── [MACHINE-TYPE]/
    ├── README.md
    ├── CHECKLIST.md
    ├── plc/
    │   ├── tia-portal/
    │   │   ├── project/           — Base .ap17 (compilable, zero errors)
    │   │   ├── export/blocks/
    │   │   ├── export/tags/
    │   │   └── README.md          — TIA version, CPU model
    │   └── studio-5000/
    │       ├── project/           — Base .ACD (compilable, zero errors)
    │       ├── export/[Name].L5X
    │       ├── export/tags/
    │       └── README.md
    ├── robot/                     — T-STD-015 and above only
    │   ├── kuka/
    │   └── yaskawa/
    ├── hmi/
    │   ├── wincc-comfort/
    │   ├── wincc-unified/         — L3 only
    │   ├── ft-view-me/
    │   └── ft-view-se/            — L3 only
    ├── docs/
    │   ├── alarms/alarm-register.xlsx
    │   ├── electrical/io-list.xlsx
    │   ├── safety/risk-assessment.xlsx
    │   ├── safety/safety-architecture.pdf
    │   ├── safety/validation-plan.xlsx
    │   ├── network/network-topology.xlsx
    │   ├── network/cybersecurity-handover.md
    │   ├── mapping/tag-xref.xlsx
    │   ├── reports/commissioning-log.xlsx
    │   └── operations/
    │       ├── operator-manual-EN.docx        — Operator Manual + Troubleshooting (master)
    │       ├── operator-manual-[LANG].docx    — Client language translation
    │       ├── training-presentation-EN.pptx
    │       ├── training-presentation-[LANG].pptx
    │       ├── competency-checklist-EN.xlsx
    │       ├── competency-checklist-[LANG].xlsx
    │       ├── operator-exam-EN.docx
    │       └── operator-exam-[LANG].docx
    └── common-modules/            — Symlinks to /common/
        └── [T-MOD-XXX] -> ../../common/[T-MOD-XXX]
```

### Template README structure

```markdown
# Template: [Machine Type]
**Template ID:** T-STD-XXX | **Version:** 1.0.0 | **Status:** ⬜/📋/🔨/✅

## Scope
[What this covers. What it does NOT cover.]

## Hardware Reference BOM
| Item | Siemens | Rockwell |
|---|---|---|
| PLC | S7-15XXF | CompactLogix LXXX |
| HMI | KPX Comfort | PanelView Plus X |

## Common Modules Used
- T-MOD-008 E-Stop
- T-MOD-001 Motor
- [...]

## Version History
| Version | Date | Notes |
|---|---|---|
| 1.0.0 | YYYY-MM | Initial |

## Known Limitations
```

---

## 8. Build Order

Ordered by count — highest reuse first. VFD comms built on-demand.

```
Phase 1 — Universal foundations (every template)
  T-MOD-008  E-Stop                    (29)
  T-MOD-019  OPC-UA map                (29)
  T-MOD-024  Digital IN                (29)
  T-MOD-025  Digital OUT               (29)
  T-MOD-001  Motor Control             (29)
  T-MOD-031  Timer Extended            (29)
  T-MOD-032  State Machine             (26)
  T-MOD-017  Alarm Buffer Profile A    (25)

Phase 2 — First standalone templates (validate Phase 1)
  T-MOD-018  Alarm Buffer Profile B    (5 — needed for L1)
  T-STD-001  Valve Cluster Small
  T-STD-004  Conveyor Single Zone

Phase 3 — High-frequency support modules
  T-MOD-002  Valve On/Off              (20)
  T-MOD-016  Shift Counter             (17)
  T-MOD-006  Counter / Encoder         (16)
  T-MOD-015  Recipe Management         (15)
  T-MOD-005  Conveyor Zone             (11) ← standalone block

Phase 4 — L2 material handling
  T-STD-002  Valve Cluster Medium
  T-STD-003  Valve Cluster Large
  T-STD-005  Conveyor Multi Zone
  T-STD-006  Sorter Simple
  T-STD-008  Piler
  T-STD-009  Pallet Dispenser
  T-STD-010  Palletizer Layer
  T-STD-011  Depalletizer
  T-STD-012  Packer / Wrapper
  T-STD-013  Flowpack
  T-STD-014  Box Former

Phase 5 — Safety + robotic (design decisions first)
  T-MOD-007  Safety Door Monitor       (design phase)
  T-MOD-009  Light Curtain             (design phase)
  T-MOD-010  Robot Interface           (design phase — unified)
  T-STD-015  Robotic Palletizer 1R
  T-STD-016  Robotic Palletizer 2R
  T-STD-017  Robotic Packing Cell
  T-STD-018  Robotic Box Former

Phase 6 — Analog + process foundations
  T-MOD-021  Analog IN                 (10)
  T-MOD-026  Scaling                   (10)
  T-MOD-023  Probe Management          (7 — TBD unified/split)
  T-MOD-022  Analog OUT                (4)
  T-MOD-004  PID                       (5)
  T-MOD-003  Valve Control             (4)
  T-MOD-027  Totalizer                 (6)
  T-MOD-028  Average                   (5)
  T-MOD-020  Process Sequence Block    (6)

Phase 7 — Process templates
  T-STD-019  Pasteurizer
  T-STD-020  Deodorizer
  T-STD-021  Mixer
  T-MOD-033  CIP Sequence
  T-STD-022  CIP Standalone
  T-STD-023  CIP Centralized
  T-STD-024  Washer

Phase 8 — Data + traceability
  T-MOD-029  FIFO / LIFO               (6)
  T-MOD-030  Lot Management            (5)

Phase 9 — Specialized / intralogistics
  T-MOD-034  Barcode / RFID
  T-MOD-035  Scale
  T-MOD-036  LGV Interface
  T-MOD-037  Servo Axis
  T-MOD-038  Tray Dispenser
  T-MOD-039  Cooling Zone
  T-STD-007  Sorter Complex
  T-STD-025  Furnace Line
  T-STD-026  RMH
  T-STD-027  LGV / AGV PLC Master
  T-STD-028  Intelligent Warehouse No Robot
  T-STD-029  Intelligent Warehouse With Robots

Phase 10 — VFD comms (on demand)
  T-MOD-011  Danfoss
  T-MOD-012  Schneider ATV
  T-MOD-013  SEW Eurodrive
  T-MOD-014  Lenze

Phase 11 — HMI/SCADA (parallel from Phase 2)
  T-HMI-001 through T-HMI-018

Phase 12 — Document templates (parallel from Phase 1)
  T-DOC-002 through T-DOC-016

Phase 13 — Operator / training documents (parallel from Phase 2, one set per machine type)
  T-DOC-017  Operator Manual + Troubleshooting
  T-DOC-018  Training Presentation
  T-DOC-019  Competency Checklist
  T-DOC-020  Operator Exam
  T-DOC-021  Technical File Index (CE — on demand)
```

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 0.4.0 | 2024-01 | Added T-DOC-017 through T-DOC-021 (operator manual, training PPT, competency checklist, exam, CE technical file). Added docs/operations/ folder to file structure. |
| 0.3.0 | 2024-01 | Module list redesigned: Motor unified, Recipe unified, VFD comms by brand on-demand, Process Sequence Block, I/O blocks, Probe TBD, FIFO/LIFO, Lot mgmt, Timer Extended, State Machine, Average, Totalizer, Scaling. Xref and build order recalculated. |
| 0.2.0 | 2024-01 | Expanded machine list, xref matrix, build order by priority |
| 0.1.0 | 2024-01 | Initial version |
