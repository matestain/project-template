# Design Document — T-STD-001 Valve Cluster Small

> Version: 0.3.0 | Status: Draft — Pending TIA Portal implementation
> Template standard: MATESTAIN company/standards/
> Platform: TIA Portal V20 | CPU: S7-1200 (S7-1214C DC/DC/DC)

---

## 1. Scope

This document is the authoritative design reference for T-STD-001 before any TIA Portal file is opened. All decisions here govern implementation. Deviations must be documented with justification.

**Machine scope:**
- Up to 15 on/off and/or open-loop modulating pneumatic valves on a single circuit
- On/off valves: single acting (spring return) and double acting
- Open-loop modulating valves: 3-position routing, direct AO output, no PID
- Auto + HMI manual operation modes
- Fixed sequence coordination (up to 30 steps, parallel valve actions per step)
- Alarm Buffer Profile B (dynamic circular list, 64 entries)
- HMI Level L1 — KTP700 Basic 6" (WinCC Basic)
- Safety minimum: PLc — Category 2, safety relay, no F-CPU required

**Out of scope:**
- Multi-circuit routing (T-STD-002/003)
- Safety valves and F-rated solenoids
- Fieldbus valve islands (Festo CPX, SMC EX600) — separate module TBD
- Hydraulic proportional valves with closed-loop position control
- More than 15 valves (T-STD-002 Medium)

**Module classification:**

| Module | Mandatory | Condition for inclusion |
|---|---|---|
| T-MOD-002 — Pneumatic Valve — On/Off | Yes | Always |
| T-MOD-008 — E-Stop Chain | Yes | Always |
| T-MOD-018a — Alarm Handler (FB_AlarmHandler) | Yes | Always |
| T-MOD-018b — Alarm Manager (FB_AlarmManager) | Yes | Always |
| T-MOD-019 — OPC-UA Data Map | Yes | Always |
| T-MOD-024 — Digital Input | Yes | Always |
| T-MOD-025 — Digital Output | Yes | Always |
| T-MOD-031 — Timer Extended | Yes | Always |
| T-MOD-032 — Sequence Runner (FB_SeqRunner) | Yes | Always |
| T-MOD-003 — Pneumatic Valve — Control | Optional | Include when ≥1 modulating valve present |
| T-MOD-001 — Motor Control | Optional | Include when pump motor present in circuit |

> **Platform coverage:** This document covers the **TIA Portal (Siemens S7-1214C)** implementation
> only. The Rockwell Studio 5000 (CompactLogix L16ER) variant will be defined in
> `DESIGN-RS5000.md` when that platform variant is built. All UDT field names, block logic,
> alarm IDs, and HMI screens are platform-agnostic and shared between variants.

---

## 2. Standard Element State Encoding

**Cross-template standard.** All field element FBs (valves, motors, actuators) compute a single
`State : INT` output using this encoding. The HMI maps this integer directly to color and text
without per-project logic. Future templates must use this encoding without modification.

| Value | Name | Meaning |
|---|---|---|
| 0 | Off | De-energized, no fault, no active command. System idle. |
| 1 | Standby | In resting position (closed/stopped), no fault, auto mode active. Ready to receive command. |
| 2 | Active | Command active and/or element in working position (open/running). |
| 3 | Alarm | Any fault active. Overrides Standby and Active. |
| 4 | Disabled | Element inhibited via `Enabled = FALSE`. Outputs de-energized, faults masked. |

**Priority rule:** 4 > 3 > 2 > 1 > 0 — always evaluate from highest priority downward.

**Standard State calculation for valve FBs:**
```
IF NOT Ctrl.Enabled THEN
    Ctrl.State := 4
ELSIF Ctrl.FltAny THEN
    Ctrl.State := 3
ELSIF EffectiveOpen OR Ctrl.StOpen OR Ctrl.StMoving THEN
    Ctrl.State := 2
ELSIF Ctrl.StClosed AND NOT Ctrl.ModeManual THEN
    Ctrl.State := 1
ELSE
    Ctrl.State := 0
END_IF
```

For motor: substitute `EffectiveOpen/StOpen/StMoving` with `EffectiveRun/StRunning/FALSE`
and `StClosed` with `StStopped`.

---

## 3. UDT Definitions

> **TIA Portal creation order** — UDTs must be created before they can be referenced. Follow this
> sequence: UDT_ValveIO → UDT_ValveCtrl → UDT_DeviceCtrl → UDT_SeqStep → UDT_SeqCtrl →
> UDT_MasterSubConfig → UDT_MasterCtrl → UDT_MotorIO → UDT_MotorCtrl →
> UDT_AlarmState → UDT_AlarmHistEntry → UDT_AlarmHistBuffer.

### 3.1 UDT_ValveIO — Physical I/O mapping (one instance per valve)

Bridge between field wiring and the valve block. Kept separate from the control UDT so the I/O map is always explicit and traceable to the I/O list. `DB_VALV_IO` must be **non-optimized** (absolute addressing enabled) so physical addresses can be mapped to its members in the tag table.

```
UDT_ValveIO:
  Outputs : STRUCT
    CmdOpen      : BOOL    // Solenoid A — open / energize
    CmdClose     : BOOL    // Solenoid B — close (double acting only)
    CmdAnalog    : INT     // AO raw value 0–27648 (modulating only; 0 for on/off)

  Inputs : STRUCT
    FbkOpen      : BOOL    // Position sensor — open / fully extended
    FbkClosed    : BOOL    // Position sensor — closed / retracted
    FbkAnalog    : INT     // AI raw value 0–27648 (modulating only; 0 for on/off)
```

`CmdAnalog` and `FbkAnalog` are raw INT (hardware counts). Scaling to 0.0–100.0% is performed inside `FB_VALV_Ctrl`. For on/off valves these members exist in the UDT but are never written or read.

### 3.2 UDT_ValveCtrl — Control and status (one instance per valve)

Control interface. Sequence runner writes commands via Dev (see Section 3.3). HMI reads status. Alarm block reads fault flags.

```
UDT_ValveCtrl:
  Config : STRUCT
    Enabled      : BOOL   // FALSE = element disabled: outputs off, faults masked (default TRUE)
    SimMode      : BOOL   // TRUE = feedback generated internally by FB — no physical signal needed (default FALSE)
    FltSeverity  : USInt  // 0=Warning 1=Hold (recoverable) 2=Abort (critical) (default: 1)
    ValveType    : INT    // 0=OnOff_Single, 1=OnOff_Double, 2=Modulating
    FbkType      : INT    // 0=None, 1=OnlyOpen, 2=OnlyClosed, 3=Binary, 4=Analog
    ToutOpen     : REAL   // Seconds allowed to reach open position (default 2.0)
    ToutClose    : REAL   // Seconds allowed to reach closed position (default 2.0)
    Discrepancy  : REAL   // Seconds before discrepancy fault triggers (default 0.5)
    SimDelay     : REAL   // Seconds after command before simulated feedback activates (default 0.5)

  Commands : STRUCT
    CmdManOpen   : BOOL   // Manual open from HMI
    CmdManClose  : BOOL   // Manual close from HMI
    CmdManual    : BOOL   // TRUE = HMI manual active, FALSE = Auto
    CmdReset     : BOOL   // Fault reset (rising edge)
    CmdResetCnt  : BOOL   // Rising edge — resets ActivationCount
    CmdManSP     : REAL   // Manual position setpoint 0.0–100.0% (modulating only; auto SP via Dev.Commands.CmdSetpoint)

  Status : STRUCT
    State        : INT    // 0=De-Energized, 1=Standby, 2=Active, 3=Alarm, 4=Disabled
    StOpen       : BOOL   // Valve is in open position
    StClosed     : BOOL   // Valve is in closed position
    StMoving     : BOOL   // Valve is in transition
    StReady      : BOOL   // Device ready to operate (enabled, no fault, in stable position)
    StPosition   : REAL   // Current position 0.0–100.0% (modulating only)
    ActivationCount : UDINT  // Open commands issued; counts rising edge of EffectiveOpen; clamped at UDINT max (4 294 967 295); persists in DB

  Faults : STRUCT
    FltAny        : BOOL  // OR of all fault flags
    FltTimeout    : BOOL  // Did not reach commanded position within ToutOpen/ToutClose
    FltDiscrepancy: BOOL  // Feedback does not match command after discrepancy time
```

> `CmdAuto` is removed — auto commands are now written by FB_SeqRunner directly to
> `DB_DEVICES.Valves[n].Commands.CmdAuto`. `FltSeverity` feeds the runner's Hold/Abort policy
> via Dev (synced each scan). `StReady` is written by the valve FB and read by the runner.

### 3.3 UDT_DeviceCtrl — Runner–Device interface (one instance per device slot in DB_DEVICES)

Adapter type that decouples the sequence runner from device-specific UDTs. FB_SeqRunner
reads and writes only `UDT_DeviceCtrl`. Device FBs (FB_VALV_OnOff, FB_MOTOR) receive both
their native UDT (`UDT_ValveCtrl` / `UDT_MotorCtrl`) and a `Dev : UDT_DeviceCtrl` IN_OUT,
and synchronize the relevant fields each scan.

```
UDT_DeviceCtrl:
  Config : STRUCT
    Enabled     : Bool    // FALSE = ignored by runner
    DevType     : USInt   // 0=Valve 1=Motor 2=Analog 3=Servo
    FltSeverity : USInt   // 0=Warning 1=Hold 2=Abort (synced from device Ctrl each scan)
  END_STRUCT

  Commands : STRUCT       // Runner writes; device FB reads
    CmdAuto     : Bool    // Valve: Open/Close — Motor: Start/Stop
    CmdSetpoint : Real    // Analog/Servo: setpoint 0.0–100.0%
  END_STRUCT

  Status : STRUCT         // Device FB writes; runner reads
    StReady     : Bool    // Device ready to operate (enabled, no fault, in stable position)
    StDone      : Bool    // Device is in ACTIVE state (Open for valves, Running for motors)
    FltAny      : Bool    // Fault active
  END_STRUCT
```

> **StDone semantics:** `StDone = TRUE` means the device is in its ACTIVE state — valve Open,
> motor Running. It does NOT mean "reached the commanded state." In COMPLETING/HOLDING/PAUSING,
> the runner commands `CmdAuto := FALSE` and waits for `StDone = FALSE` (all closed/stopped).
> CondTarget bit=1 → advance when StDone=TRUE (device active); bit=0 → advance when StDone=FALSE.

### 3.4 UDT_SeqStep — Sequence step descriptor

Mask-based, polymorphic by StepType. Bits 0–14 correspond to devices 1–15 in the runner's Devices array. `ActSetpoint[1..15]` is T-STD-001 specific (reference uses [1..32]; adapted to max 15 valves to save ~1.8 KB per 30-step program).

```
UDT_SeqStep:
  StepType    : USInt          // 0=Action 1=TimedWait 2=ExternalCond 3=SubSeq
  Timeout     : REAL           // Max seconds for this step (0.0 = no timeout)
  ActMask     : DWORD          // Bit n=1 → command device (n+1) in this step
  ActTarget   : DWORD          // Bit n=1 → CmdAuto=TRUE (open/run); 0 → CmdAuto=FALSE (close/stop)
  ActSetpoint : ARRAY[1..15] OF REAL   // Setpoint per device slot (modulating valves only)
  CondMask    : DWORD          // Bit n=1 → evaluate StDone of device (n+1) before advancing
  CondTarget  : DWORD          // Bit n=1 → advance when StDone=TRUE; 0 → advance when StDone=FALSE
  WaitTime    : REAL           // Seconds to wait (StepType=1)
  ExtCondIdx  : USInt          // Index into ExtCond[] (StepType=2, 1..8)
  SubSeqIdx   : USInt          // Sub-sequence index (StepType=3, 1..8)
  Parallel    : Bool           // TRUE = advance to next step immediately (don't wait for this step)
  StepDesc    : STRING[32]     // Human-readable description for HMI diagnostics
```

**Example — Open V01 and V03 simultaneously, wait for both (StepType=0):**
```
StepType   := USInt#0
ActMask    := DWORD#16#0000_0005   // bits 0 and 2 (devices 1 and 3)
ActTarget  := DWORD#16#0000_0005   // both commanded open
CondMask   := DWORD#16#0000_0005   // wait for feedback from both
CondTarget := DWORD#16#0000_0005   // advance condition = StDone=TRUE (open) for both
Timeout    := 5.0
StepDesc   := 'Open V01 and V03'
```

**Example — Close V01 only, no feedback wait (FbkType=0 valve):**
```
StepType   := USInt#0
ActMask    := DWORD#16#0000_0001   // bit 0 = device 1
ActTarget  := DWORD#16#0000_0000   // command close
CondMask   := DWORD#16#0000_0000   // no wait — advance immediately
StepDesc   := 'Close V01'
```

**Example — Timed wait 3 seconds (StepType=1):**
```
StepType   := USInt#1
WaitTime   := 3.0
StepDesc   := 'Wait 3s for pressure to stabilize'
```

### 3.5 UDT_SeqCtrl — Sequence runner control (one instance per FB_SeqRunner)

Full PackML / OMAC ISA-88 structure. Replaces the v0.2.0 UDT_ClusterCtrl.

```
UDT_SeqCtrl:
  Config : STRUCT
    AutoResume      : Bool    // TRUE = auto-resume from HELD if fault clears
    AutoResumeTime  : Real    // Max seconds for auto-resume (0.0 = no time limit)
    StopTimeout     : Real    // Seconds to confirm closure in STOPPING/ABORTING (default 5.0)
    HoldTimeout     : Real    // Seconds in HOLDING before going to ABORTING (default 30.0)
  END_STRUCT

  Commands : STRUCT
    CmdStart    : Bool
    CmdStop     : Bool
    CmdPause    : Bool
    CmdResume   : Bool
    CmdAbort    : Bool
    CmdReset    : Bool
    CmdClear    : Bool
  END_STRUCT

  Status : STRUCT
    State       : USInt   // PackML state value (see constants in FB_SeqRunner)
    StStep      : Int     // Current step index (1..nSteps; 1 in IDLE)
    StIdle      : Bool    // State = IDLE (0)
    StStarting  : Bool    // State = STARTING (1)
    StExecute   : Bool    // State = EXECUTE (2)
    StCompleting: Bool    // State = COMPLETING (3)
    StComplete  : Bool    // State = COMPLETE (4)
    StHolding   : Bool    // State = HOLDING (5)
    StHeld      : Bool    // State = HELD (6)
    StResuming  : Bool    // State = RESUMING (7)
    StPausing   : Bool    // State = PAUSING (8)
    StPaused    : Bool    // State = PAUSED (9)
    StStopping  : Bool    // State = STOPPING (11)
    StStopped   : Bool    // State = STOPPED (12)
    StAborting  : Bool    // State = ABORTING (13)
    StAborted   : Bool    // State = ABORTED (14)
    StDone      : Bool    // Sequence completed all steps cleanly (set in COMPLETE)
  END_STRUCT

  Faults : STRUCT
    FaultCode   : USInt   // 0=None 1=StepTimeout 2=EStop/Abort 3=DevFault 4=HoldTimeout
    FaultStep   : Int     // Step index where fault occurred
    FaultDevice : Int     // Device index that caused fault (1..15; 0=none)
    FltAny      : Bool
  END_STRUCT

  Interlocks : STRUCT     // Written by OB1 inline logic before calling FB_SeqRunner
    IntlkReady  : Bool    // Process preconditions met (air OK, no motor fault)
    IntlkEstop  : Bool    // E-Stop chain OK (TRUE = safe to operate)
  END_STRUCT
```

### 3.6 UDT_MasterSubConfig — Master policy per sub-sequence

```
UDT_MasterSubConfig:
  OnFaultPolicy   : USInt   // 0=AbortAll 1=AbortOnly 2=HoldAll 3=HoldOnly
  OnTimeoutPolicy : USInt   // 0=AbortAll 1=AbortOnly 2=HoldAll 3=HoldOnly
  IsMandatory     : Bool    // FALSE = fault in this runner does not affect others
```

### 3.7 UDT_MasterCtrl — Master runner control (one instance per FB_SeqMaster)

```
UDT_MasterCtrl:
  Config : STRUCT
    SubPolicy   : ARRAY[1..8] OF UDT_MasterSubConfig
  END_STRUCT

  Commands : STRUCT
    CmdStart    : Bool
    CmdStop     : Bool
    CmdPause    : Bool
    CmdResume   : Bool
    CmdAbort    : Bool
    CmdReset    : Bool
  END_STRUCT

  Status : STRUCT
    State       : USInt
    StIdle      : Bool
    StExecute   : Bool
    StHeld      : Bool
    StPaused    : Bool
    StStopped   : Bool
    StAborted   : Bool
    StComplete  : Bool
    StDone      : Bool
  END_STRUCT

  Faults : STRUCT
    FaultCode   : USInt
    FaultSubSeq : USInt   // Sub-sequence index that caused the fault
    FltAny      : Bool
  END_STRUCT

  Interlocks : STRUCT
    IntlkReady  : Bool
    IntlkEstop  : Bool
  END_STRUCT
```

### 3.8 UDT_MotorIO — Motor physical I/O (optional — include only if pump motor present)

Same pattern as `UDT_ValveIO`: all physical field connections live in the I/O UDT.
`DB_MOTOR_IO` must be **non-optimized** (absolute addressing enabled).

```
UDT_MotorIO:
  Outputs : STRUCT
    CmdRun       : BOOL    // → contactor coil %Q

  Inputs : STRUCT
    FbkRunning   : BOOL    // Aux contact — motor is running %I
    FbkOverload  : BOOL    // Thermal relay — TRUE = OK, FALSE = tripped (NC wired) %I
```

### 3.9 UDT_MotorCtrl — Motor control and status

```
UDT_MotorCtrl:
  Config : STRUCT
    Enabled      : BOOL   // FALSE = element disabled: outputs off, faults masked (default TRUE)
    SimMode      : BOOL   // TRUE = feedback generated internally by FB (default FALSE)
    FltSeverity  : USInt  // 0=Warning 1=Hold 2=Abort [default: 2 — motor overload = abort]
    ToutRun      : REAL   // Timeout for run confirmation in seconds (default 5.0)
    SimDelay     : REAL   // Seconds after CmdAuto before simulated FbkRunning activates (default 1.0)

  Commands : STRUCT
    CmdManRun         : BOOL   // Manual run from HMI (manual mode only)
    CmdManStop           : BOOL   // Stop command from HMI
    CmdManual    : BOOL   // TRUE = HMI manual active, FALSE = Auto
    CmdReset     : BOOL   // Fault reset (rising edge)
    CmdResetCnt  : BOOL   // Rising edge — resets ActivationCount

  Status : STRUCT
    State        : INT    // 0=De-Energized, 1=Standby, 2=Active, 3=Alarm, 4=Disabled
    StRunning    : BOOL   // Motor is confirmed running
    StStopped    : BOOL   // Motor is confirmed stopped
    StReady      : BOOL   // Enabled AND NOT FltAny AND StStopped
    ActivationCount : UDINT  // Start commands issued; counts rising edge of EffectiveRun; clamped at UDINT max; persists in DB
    RunHours        : REAL   // Accumulated confirmed running hours (StRunning=TRUE); clamped at 999 999.9 h; persists in DB

  Faults : STRUCT
    FltAny       : BOOL   // Fault Active
    FltTimeout   : BOOL   // Run confirmation not received within ToutRun
    FltOverload  : BOOL   // Thermal/overload relay tripped
```

> `CmdRun` is removed — auto run command is now written by FB_SeqRunner to
> `DB_DEVICES.Motors[n].Commands.CmdAuto`. Same pattern as valves.

### 3.10 UDT_AlarmState — Active state of a single alarm

Replaces v0.2.0 `UDT_AlarmEntry`. Direct O(1) access — one fixed slot per alarm in `DB_ALARMS`.

```
UDT_AlarmState:
  AlarmID      : DInt    // Encoded: F=1 W=2 I=3 × 100000 + Domain × 10000 + Index
  Active       : Bool    // Alarm is currently active
  Acknowledged : Bool    // Operator has acknowledged
  TimeRaised   : DTL     // System time at last activation
  TimeCleared  : DTL     // System time at last deactivation
  Count        : Int     // Activations since last buffer clear
  Severity     : USInt   // 0=Info 1=Warning 2=Error 3=Critical
  Category     : USInt   // 1=Valves 2=System (T-STD-001 categories)
  Device       : USInt   // Valve index 1..15; 0=system-level alarm
```

**AlarmID encoding:**
```
AlarmID := DInt(SeverityDigit × 100000 + DomainDigit × 10000 + Index)
// SeverityDigit: F=1, W=2, I=3
// Examples:
//   F1-0001 → 1×100000 + 1×10000 + 1   = 110001
//   F1-0031 → 1×100000 + 1×10000 + 31  = 110031
//   W1-0001 → 2×100000 + 1×10000 + 1   = 210001
//   I1-0001 → 3×100000 + 1×10000 + 1   = 310001
//   F9-0001 → 1×100000 + 9×10000 + 1   = 190001
```

> Type is `DInt` (not `DWORD`) — matches TIA Portal SCL literal syntax (`DInt#110001`).
> F and W now have distinct first digits, eliminating the v0.2.0 collision.

### 3.11 UDT_AlarmHistEntry — One event record in the history ring buffer

```
UDT_AlarmHistEntry:
  AlarmID      : DInt
  Acknowledged : Bool
  Valid        : Bool    // FALSE = empty slot in ring buffer
  TimeRaised   : DTL
  TimeCleared  : DTL
  Severity     : USInt
  Category     : USInt
  Device       : USInt
```

### 3.12 UDT_AlarmHistBuffer — Ring buffer wrapper (Profile B: 64 entries)

Replaces v0.2.0 `UDT_AlarmBuffer`. Array sized to Profile B (64 entries). To change capacity,
adjust the array bounds and set `MaxSize` to the same value in OB100.

```
UDT_AlarmHistBuffer:
  Head     : Int     // Next write index
  Tail     : Int     // Next read index (external consumer)
  Count    : Int     // Valid entries currently in buffer
  MaxSize  : Int     // Maximum size — must equal array bound + 1 (set := 64 in OB100)
  Full     : Bool    // TRUE = buffer full, next write overwrites oldest
  Entries  : ARRAY[0..63] OF UDT_AlarmHistEntry
```

---

## 4. Global DB Structure

> **Rule — global vs instance DBs:** All data read by the HMI, accessed by OB1 from outside
> the block, or shared between blocks lives in a **global DB**. Instance DBs contain ONLY
> internal FB state (timers, edge-detection booleans, counters). Never bind an HMI element
> to an instance DB member, and never read an instance DB member from a block that didn't
> create it.

All shared data lives in named global DBs, not in PLC tag tables. Non-optimized DBs
(absolute addressing enabled) are required only for those that map to physical I/O.
Optimized access is required for OPC-UA efficiency on `DB_ALARMS` and `DB_HIST`.

```
DB_VALV_IO      : Global DB [non-optimized]    DB9010
  IO : ARRAY[1..15] OF UDT_ValveIO

DB_VALV_CTRL    : Global DB [optimized]        DB9011
  CTRL : ARRAY[1..15] OF UDT_ValveCtrl

DB_SEQUENCES    : Global DB [optimized]        DB9012
  RunnerCtrl : UDT_SeqCtrl
  SeqSteps   : ARRAY[1..30] OF UDT_SeqStep
  nSteps     : INT

DB_SYS          : Global DB [optimized]        DB9013
  EStop           : UDT_EStopCtrl
  Motor           : UDT_MotorCtrl    // Present always; Enabled=FALSE if no pump motor
  AirOK           : BOOL             // Mapped from physical DI via FB_DI
  AlarmCmdAckAll  : BOOL             // Written by OPC-UA/HMI → passed to FB_AlarmManager
  AlarmCmdClear   : BOOL             // Written by OPC-UA (supervisor only)
  WdgSnapshot     : DINT             // Watchdog snapshot (current scan)
  WdgPrev         : DINT             // Watchdog snapshot (previous scan)

DB_ALARMS       : Global DB [optimized]        DB9014
  Valves : ARRAY[0..29] OF UDT_AlarmState   // Slots: Valve n → [2n-2]=timeout, [2n-1]=discrepancy
  System : ARRAY[0..6]  OF UDT_AlarmState   // [0]=F1-0031 [1]=F1-0032 [2]=W1-0001 [3]=W1-0002
                                             // [4]=I1-0001 [5]=I1-0002 [6]=F9-0001

DB_ALARM_HIST         : Global DB [optimized]        DB9015
  Buf : UDT_AlarmHistBuffer

DB_MOTOR_IO     : Global DB [non-optimized]    DB9016
  IO : UDT_MotorIO                   // Single motor instance for T-STD-001

DB_DEVICES      : Global DB [optimized]        DB9017
  Valves : ARRAY[1..32] OF UDT_DeviceCtrl   // Positions 16..32: Enabled=FALSE for T-STD-001
  Motors : ARRAY[1..16] OF UDT_DeviceCtrl   // Position [1] for pump; rest Enabled=FALSE
```

DB numbers follow the T-STD-001 global DB range (DB9010–DB9019) per `naming-conventions.md` Section 8.3.

**DB_DEVICES purpose:** Adapter between FB_SeqRunner (which operates on `ARRAY[1..32] OF UDT_DeviceCtrl`)
and device-specific UDTs (UDT_ValveCtrl, UDT_MotorCtrl). Device FBs receive `Dev : UDT_DeviceCtrl`
as an IN_OUT parameter and synchronize relevant fields each scan (see Sections 5–6, 8.4).

**I/O copy pattern:** At the top of OB1, copy physical input tags to `DB_VALV_IO.IO[n]`
input members. At the bottom of OB1, copy `DB_VALV_IO.IO[n]` output members to physical
output tags. This keeps all logic working against the DB and makes the I/O map entirely
visible in one place.

---

## 5. Block Design — FB_VALV_OnOff (T-MOD-002)

### 5.1 Interface

On S7-1200 (TIA Portal V20, firmware V4.x+), `IN_OUT` with a UDT passes by reference — no copy overhead. `REF_TO` (LREF) is an S7-1500-only feature and must not be used here.

```
FB_VALV_OnOff                                          // FB200
  IN_OUT:
    IO   : UDT_ValveIO    // Physical I/O — pass DB_VALV_IO.IO[n]
    Ctrl : UDT_ValveCtrl  // Control and status — pass DB_VALV_CTRL.CTRL[n]
    Dev  : UDT_DeviceCtrl // Runner interface — pass DB_DEVICES.Valves[n]
```

### 5.2 Internal variables (static, in instance DB)

```
  tToutOpen     : IEC_TIMER  // Timeout timer — open direction
  tToutClose    : IEC_TIMER  // Timeout timer — close direction
  tDiscrepancy  : IEC_TIMER  // Discrepancy timer
  tSimOpen      : IEC_TIMER  // Sim feedback timer — open direction
  tSimClose     : IEC_TIMER  // Sim feedback timer — close direction
  xLastRst      : BOOL       // Last CmdReset state for edge detection
  xlastEffOpen  : BOOL       // Last EffectiveOpen state for activation counter edge detection
  xlastResetCnt : BOOL       // Last CmdResetCounters state for edge detection
```

TEMP variables (recalculated each scan, not stored):
```
  xEffectiveOpen       : BOOL  // Resolved open command (manual or auto)
  xEffectiveClose      : BOOL  // Resolved close command (double acting only)
  xEffectiveFbkOpen    : BOOL  // Effective open feedback (real or simulated)
  xEffectiveFbkClosed  : BOOL  // Effective closed feedback (real or simulated)
  xDiscrepancy         : BOOL  // Instantaneous discrepancy condition
```

### 5.3 Logic description

Language: LAD.
Note: `RETURN` behavior is implemented in LAD by setting a `bEnabled` series contact that disables
all subsequent rungs when `Ctrl.Enabled = FALSE`. The IF NOT Enabled → RETURN pattern below
describes the logical intent.

**Config sync (before Step 0 — first rungs, always executed):**
```scl
Dev.Config.DevType     := USInt#0;         // Valve
Dev.Config.Enabled     := Ctrl.Config.Enabled;
Dev.Config.FltSeverity := Ctrl.Config.FltSeverity;
```

**Step 0 — Disabled check (evaluate first):**
```
IF NOT Ctrl.Config.Enabled THEN
    IO.Outputs.CmdOpen  := FALSE
    IO.Outputs.CmdClose := FALSE
    Ctrl.Status.State  := 4
    JMP END  // Skip all remaining steps (bEnabled := FALSE disables subsequent rungs in LAD)
END_IF
```

**Step 1 — Mode resolution:**
```
IF Ctrl.Commands.CmdManual THEN
    xEffectiveOpen  := Ctrl.Commands.CmdManOpen
    xEffectiveClose := (Ctrl.Commands.CmdManClose AND Ctrl.Config.ValveType = 2) // Double acting only
ELSE
    xEffectiveOpen  := Dev.Commands.CmdAuto              // Runner writes this each scan
    xEffectiveClose := NOT Dev.Commands.CmdAuto          // Single acting: spring closes
END_IF
```

**Step 2 — SimMode feedback generation:**
```
tSimOpen.PT  := Ctrl.Config.SimDelay * 1000   //Conversion is automatic with MUL
tSimClose.PT := tSimOpen.PT

IF Ctrl.Config.SimMode THEN
    tSimOpen.IN  := xEffectiveOpen
    tSimClose.IN := NOT xEffectiveOpen OR (xEffectiveClose AND Ctrl.Config.ValveType = 2)

    xEffectiveFbkOpen   := tSimOpen.Q    // Simulated: open confirmed after SimDelay
    xEffectiveFbkClosed := tSimClose.Q   // Simulated: closed confirmed after SimDelay
ELSE
    xEffectiveFbkOpen   := IO.Inputs.FbkOpen    // Real physical feedback
    xEffectiveFbkClosed := IO.Inputs.FbkClosed
END_IF
```

Physical outputs (IO.Outputs.CmdOpen, IO.Outputs.CmdClose) are still written to hardware in SimMode — this allows solenoid wiring verification during FAT. Only the feedback path is intercepted.

**Step 3 — Output to field (fault locks outputs to safe state):**
```
IO.Outputs.CmdOpen  := xEffectiveOpen  AND NOT Ctrl.Faults.FltAny
IO.Outputs.CmdClose := xEffectiveClose AND NOT Ctrl.Faults.FltAny AND Ctrl.Config.ValveType = 2   // Double acting only
```

**Step 4 — Position status (FbkType enum: 0=None, 1=OnlyOpen, 2=OnlyClosed, 3=Binary, 4=Analog):**
SCL Justified (1 simple NW)
```
CASE Ctrl.Config.FbkType OF
  0:  // No feedback
    Ctrl.Status.StOpen   := IO.Outputs.CmdOpen;
    Ctrl.Status.StClosed := IO.Outputs.CmdClose OR NOT IO.Outputs.CmdOpen;
    Ctrl.Status.StMoving := FALSE;

  1:  // Only open sensor
    Ctrl.Status.StOpen   := xEffectiveFbkOpen;
    Ctrl.Status.StClosed := NOT xEffectiveFbkOpen AND NOT xEffectiveOpen;
    Ctrl.Status.StMoving := xEffectiveOpen AND NOT xEffectiveFbkOpen;

  2:  // Only closed sensor
    Ctrl.Status.StOpen   := NOT xEffectiveFbkClosed AND xEffectiveOpen;
    Ctrl.Status.StClosed := xEffectiveFbkClosed;
    Ctrl.Status.StMoving := NOT xEffectiveFbkClosed AND xEffectiveOpen;

  3:  // Binary (open + closed sensors)
    Ctrl.Status.StOpen   := xEffectiveFbkOpen;
    Ctrl.Status.StClosed := xEffectiveFbkClosed;
    Ctrl.Status.StMoving := NOT xEffectiveFbkOpen AND NOT xEffectiveFbkClosed;
END_CASE;
```

**Step 5 — Timeout fault (FbkType ≠ 0 only; separate timers for open and close):**
```
tToutOpen.PT  := Ctrl.Config.ToutOpen * 1000     //Conversion is automatic with MUL
tToutClose.PT  := Ctrl.Config.ToutClose * 1000   //Conversion is automatic with MUL

tToutOpen.IN  := xEffectiveOpen  AND NOT Ctrl.Status.StOpen  AND (Ctrl.Config.FbkType <> 0)
tToutClose.IN := xEffectiveClose AND NOT Ctrl.Status.StClosed AND (Ctrl.Config.FbkType <> 0)

IF tToutOpen.Q OR tToutClose.Q THEN
    Ctrl.Faults.FltTimeout := TRUE     //SET
END_IF
```

**Step 6 — Discrepancy fault (FbkType = 3 only; steady-state check):**
```
tDiscrepancy.PT  := Ctrl.Config.Discrepancy * 1000     //Conversion is automatic with MUL

xDiscrepancy := (IO.Commands.CmdOpen AND xEffectiveFbkClosed) OR (NOT IO.Commands.CmdOpen AND xEffectiveFbkOpen) OR 
                (IO.Commands.CmdClose AND xEffectiveFbkOpen) OR (NOT IO.Commands.CmdClose AND xEffectiveFbkClose)
                AND NOT Ctrl.Status.StMoving

tDiscrepancy.IN := xDiscrepancy AND (Ctrl.Config.FbkType = 3)

IF tDiscrepancy.Q THEN
    Ctrl.Faults.FltDiscrepancy := TRUE  //SET
END_IF
```

**Step 7 — Fault aggregation and reset (rising edge on CmdReset):**
```
Ctrl.Faults.FltAny := Ctrl.Faults.FltTimeout OR Ctrl.Faults.FltDiscrepancy

IF Ctrl.Faults.FltAny THEN
    Dev.Commands.CmdAuto         := FALSE
    Ctrl.Commands.CmdManOpen     := FALSE
    Ctrl.Commands.CmdManClose    := FALSE
END_IF

IF Ctrl.Commands.CmdReset AND NOT xLastReset THEN
    Ctrl.Faults.FltTimeout     := FALSE
    Ctrl.Faults.FltDiscrepancy := FALSE
END_IF
xLastReset := Ctrl.Commands.CmdReset
// Reset clears flags only — sequence must re-issue the command
```

**Step 8 — State calculation (Section 2 standard encoding):**
```SCL Justified (1 simple NW)
IF NOT Ctrl.Config.Enabled THEN
    Ctrl.Status.State := 4;
ELSIF Ctrl.Faults.FltAny THEN
    Ctrl.Status.State := 3;
ELSIF xEffectiveOpen OR Ctrl.Status.StOpen OR Ctrl.Status.StMoving THEN
    Ctrl.Status.State := 2;
ELSIF Ctrl.Status.StClosed AND NOT Ctrl.Commands.CmdManual THEN
    Ctrl.Status.State := 1;
ELSE
    Ctrl.Status.State := 0;
END_IF;
```

**Step 9 — StReady calculation and Dev.Status sync:**
```SCL Justified (1 simple NW)
Ctrl.Status.StReady := Ctrl.Config.Enabled AND NOT Ctrl.Faults.FltAny AND (Ctrl.Status.StOpen OR Ctrl.Status.StClosed OR (Ctrl.Config.FbkType = 0));

Dev.Status.StDone  := Ctrl.Status.StOpen;    // Active state = Open
Dev.Status.StReady := Ctrl.Status.StReady;
Dev.Status.FltAny  := Ctrl.Faults.FltAny;
```

**Step 10 — Activation counter:**
```SCL Justified (math)
// Rising edge xEffectiveOpen → increment (clamped to prevent UDINT overflow)
IF xEffectiveOpen AND NOT xlastEffectiveOpen THEN
    IF Ctrl.Status.ActivationCnt < 4294967295 THEN
        Ctrl.Status.ActivationCnt := Ctrl.Status.ActivationCnt + 1;
    END_IF;
END_IF;
xlastEffectiveOpen := xEffectiveOpen;

// Rising edge CmdResetCnt → clear counter
IF Ctrl.Commands.CmdResetCnt AND NOT xlastResetCnt THEN
    Ctrl.Status.ActivationCnt := 0;
END_IF;
xlastResetCnt := Ctrl.Commands.CmdResetCnt;
```

**END Flag**

### 5.4 Safe state behavior

On fault: both outputs de-energize (`IO.Commands.CmdOpen := FALSE`, `IO.Commands.CmdClose := FALSE`).
- Single acting: spring returns to rest position (typically closed — verify per P&ID).
- Double acting: de-energizing both solenoids holds last position. **Document the safe state for each double acting valve in the I/O list** — some installations require forced close on fault (connect via hardware interlock).

---

## 6. Block Design — FB_VALV_Ctrl (T-MOD-003)

Open-loop modulating valve — 3-position routing or direct AO control. No PID. `FbkAnalog` and `CmdAnalog` are raw INT (hardware counts 0–27648 for 4-20mA).

### 6.1 Interface

```
FB_VALV_Ctrl                                           // FB210
  IN_OUT:
    IO   : UDT_ValveIO    // Physical I/O — pass DB_VALV_IO.IO[n]
    Ctrl : UDT_ValveCtrl  // Control and status — pass DB_VALV_CTRL.CTRL[n]
    Dev  : UDT_DeviceCtrl // Runner interface — pass DB_DEVICES.Valves[n]
```

### 6.2 Internal variables (static, in instance DB)

```
  tTout          : IEC_TIMER  // Position reach timeout
  rLastSP           : REAL    // Last setpoint for change detection
  xLastReset        : BOOL    // Last CmdReset state for edge detection
  xlastEffectiveSP  : BOOL    // Last (EffectiveSP > 5.0) state for activation counter edge detection
  xlastResetCnt     : BOOL    // Last CmdResetCounters state for edge detection
```

TEMP variables:
```
  rEffectiveSP  : REAL   // Resolved setpoint (manual or auto)
  rScratch      : REAL
  xNotAtSP      : BOOL   // Position not within 5% of setpoint
  xEffectiveSP  : BOOL   // EffectiveSP > 5.0 — used for activation counter edge detection
```

### 6.3 Logic description

Language: LAD.

**Config sync (before Step 0):**
```scl
Dev.Config.DevType     := USInt#0;
Dev.Config.Enabled     := Ctrl.Enabled;
Dev.Config.FltSeverity := Ctrl.FltSeverity;
```

**Step 0 — Disabled check:**
```
IF NOT Ctrl.Config.Enabled THEN
    IO.Commands.CmdAnalog := 0
    Ctrl.Status.State   := 4
    JMP END             // Skip remaining steps
END_IF
```

**Step 1 — Mode resolution:**
```
IF Ctrl.ModeManual THEN
    rEffectiveSP := Ctrl.Commands.CmdSP            // Manual setpoint 0.0–100.0% from HMI
ELSE
    rEffectiveSP := Dev.Commands.CmdSetpoint       // Runner setpoint 0.0–100.0%
END_IF
```

**Step 2 — AO output (scale 0.0–100.0% → 0–27648 raw INT):**
```
IO.Commands.CmdAnalog := rEffectiveSP * 276.48    // 100% × 276.48 = 27648  // Conversion is automatic with MUL
```

**Step 3 — Position feedback and SimMode:**
```
IF NOT Ctrl.Config.SimMode THEN
    Ctrl.Status.StPosition := IO.Inputs.FbkAnalog / 276.48  // Conversion is automatic with DIV
    xNotAtSP  := ABS(Ctrl.Status.StPosition - rEffectiveSP) > 5.0    // rScratch is used here as aux tag
ELSE
    Ctrl.Status.StPosition := rEffectiveSP    // Simulate instantaneous position = setpoint
END_IF

Ctrl.Status.StOpen   := Ctrl.Status.StPosition >= 95.0
Ctrl.Status.StClosed := Ctrl.Status.StPosition <= 5.0
Ctrl.Status.StMoving := NOT Ctrl.Status.StOpen AND NOT Ctrl.Status.StClosed
```

**Step 4 — Timeout fault (position not within 5% of setpoint, SimMode disabled):**
```
tTimeout.PT := Ctrl.Config.ToutOpen * 1000  // Conversion is automatic with MUL

tTimeout.IN := xNotAtSP
Ctrl.Faults.FltTimeout := tTimeout.Q
```

**Step 5 — Fault lock and reset:**
```
Ctrl.Faults.FltAny := Ctrl.Faults.FltTimeout

IF Ctrl.Faults.FltAny THEN
    Dev.Commands.CmdAnalog := 0
END_IF

IF Ctrl.Commands.CmdReset AND NOT xLastReset THEN
    Ctrl.Faults.FltTimeout := FALSE
END_IF
xLastReset := Ctrl.Commands.CmdReset
```

On fault: `IO.CmdAnalog := 0` (valve drives to 0% — verify safe position per P&ID).

**Step 6 — State calculation:**
```scl
IF NOT Ctrl.Config.Enabled THEN
    Ctrl.Status.State := 4;
ELSIF Ctrl.Faults.FltAny THEN
    Ctrl.Status.State := 3;
ELSIF rEffectiveSP > 5.0 OR Ctrl.Status.StOpen OR Ctrl.Status.StMoving THEN
    Ctrl.Status.State := 2;
ELSIF Ctrl.Status.StClosed AND NOT Ctrl.Commands.CmdManual THEN
    Ctrl.Status.State := 1;
ELSE
    Ctrl.Status.State := 0;
END_IF;
```

**Step 7 — StReady and Dev.Status sync:**
```scl
Ctrl.Status.StReady := Ctrl.Config.Enabled AND NOT Ctrl.Faults.FltAny AND (Ctrl.Status.StOpen OR Ctrl.Status.StClosed);

Dev.Status.StDone  := Ctrl.Status.StOpen;
Dev.Status.StReady := Ctrl.Status.StReady;
Dev.Status.FltAny  := Ctrl.Faults.FltAny;
```

**Step 8 — Activation counter:**
```scl
xEffectiveSP := rEffectiveSP > 5.0;

// Rising edge (setpoint enters active range) → increment (clamped to prevent UDINT overflow)
IF xEffectiveSP AND NOT xlastEffectiveSP THEN
    IF Ctrl.Status.ActivationCnt < 4294967295 THEN
        Ctrl.Status.ActivationCnt := Ctrl.Status.ActivationCnt + 1;
    END_IF;
END_IF;
xlastEffectiveSP := xEffectiveSP;

// Rising edge CmdResetCnt → clear counter
IF Ctrl.Commands.CmdResetCnt AND NOT xlastResetCnt THEN
    Ctrl.Status.ActivationCnt := 0;
END_IF;
xlastResetCnt := Ctrl.Commands.CmdResetCnt;
```

---

## 7. Block Design — FB_SeqRunner and FB_SeqMaster (T-MOD-032)

### 7.1 FB_SeqRunner (FB300) — PackML Standalone Sequence Runner

Language: **SCL** — justified: 14-state PackML machine with FOR loops over device arrays,
bitwise DWORD mask operations, and multiple TON timers.

**Interface:**
```
FB_SeqRunner (SCL)                                     // FB300
  IN:
    nDevices    : INT    // Actual number of devices in use (1..15 for T-STD-001)
    nSteps      : INT    // Number of steps — pass DB_SEQUENCES.nSteps
  IN_OUT:
    Ctrl        : UDT_SeqCtrl                    // Pass DB_SEQUENCES.RunnerCtrl
    Devices     : ARRAY[1..32] OF UDT_DeviceCtrl // Pass DB_DEVICES.Valves
    Steps       : ARRAY[1..30] OF UDT_SeqStep    // Pass DB_SEQUENCES.SeqSteps
    ExtCond     : ARRAY[1..8]  OF Bool           // External advance conditions (stubs in T-STD-001)
    SubDone     : ARRAY[1..8]  OF Bool           // Sub-sequence done feedback (stub)
    SubStart    : ARRAY[1..8]  OF Bool           // Sub-sequence start command (stub)
    SubHeld     : ARRAY[1..8]  OF Bool           // Sub-sequence held feedback (stub)
    SubPaused   : ARRAY[1..8]  OF Bool           // Sub-sequence paused feedback (stub)
  Static:
    tStopTout       : TON_TIMER
    tStepTout       : TON_TIMER
    tWait           : TON_TIMER
    tHoldTout       : TON_TIMER
    tAutoResume     : TON_TIMER
    iFaultDevice    : Int
    iFaultSeverity  : USInt
    xHoldFromPause  : Bool
    xStepDone       : Bool
    xAllSafe        : Bool
    xLastStart      : Bool
    xLastStop       : Bool
    xLastPause      : Bool
    xLastResume     : Bool
    xLastAbort      : Bool 
    xLastReset      : Bool
  Temp:
    n               : Int
    dBitVal         : DWord

```

**PackML state constants:**

| State | Value |
|---|---|
| ST_IDLE | 0 |
| ST_STARTING | 1 |
| ST_EXECUTE | 2 |
| ST_COMPLETING | 3 |
| ST_COMPLETE | 4 |
| ST_HOLDING | 5 |
| ST_HELD | 6 |
| ST_RESUMING | 7 |
| ST_PAUSING | 8 |
| ST_PAUSED | 9 |
| ST_STOPPING | 11 |
| ST_STOPPED | 12 |
| ST_ABORTING | 13 |
| ST_ABORTED | 14 |

**State machine logic summary:**

Priority: `CmdAbort` or `IntlkEstop = FALSE` → immediate transition to ST_ABORTING (any state except ST_ABORTED).

```
ST_IDLE:
    Clear commands to all devices (CmdAuto := FALSE)
    CmdStart + IntlkReady + IntlkEstop → ST_STARTING

ST_STARTING:
    Check all enabled devices: FltAny=FALSE AND StReady=TRUE
    If all OK → ST_EXECUTE (StStep := 1)
    If IntlkEstop=FALSE → ST_ABORTING

ST_EXECUTE:
    Check device faults:
        FltSeverity=1 (Hold) → ST_HOLDING; FaultCode:=3
        FltSeverity=2 (Abort) → ST_ABORTING; FaultCode:=3
    Execute Steps[StStep] by StepType:
        0=Action: apply ActMask/ActTarget; evaluate CondMask/CondTarget (StDone bits)
        1=Wait: _tWait runs WaitTime seconds
        2=ExternalCond: check ExtCond[ExtCondIdx]
        3=SubSeq: set SubStart[SubSeqIdx]; done when SubDone[SubSeqIdx] (or Parallel=TRUE)
    Step timeout: _tStepTout → ST_ABORTING; FaultCode:=1
    Step advance: StStep < nSteps → StStep+1; at last step → ST_COMPLETING
    CmdPause → ST_PAUSING; CmdStop → ST_STOPPING

ST_COMPLETING:
    CmdAuto:=FALSE, CmdSetpoint:=0.0 for all enabled devices
    Wait for all enabled devices: StDone=FALSE (all closed/stopped)
    All safe → ST_COMPLETE; StDone:=TRUE

ST_COMPLETE:
    CmdReset → ST_IDLE

ST_HOLDING:
    CmdAuto:=FALSE, CmdSetpoint:=0.0 for all (preserve StStep)
    Wait StDone=FALSE for all; HoldTimeout → ST_ABORTING; FaultCode:=4
    All safe → ST_HELD

ST_HELD:
    CmdResume → ST_RESUMING
    AutoResume: if all FltAny=FALSE for AutoResumeTime → ST_RESUMING; clear faults
    CmdStop → ST_STOPPING

ST_RESUMING:
    All enabled devices: FltAny=FALSE AND StReady=TRUE → ST_EXECUTE; clear faults

ST_PAUSING:
    CmdAuto:=FALSE for all; wait StDone=FALSE → ST_PAUSED

ST_PAUSED:
    CmdResume → ST_RESUMING; CmdStop → ST_STOPPING

ST_STOPPING:
    CmdAuto:=FALSE for all; wait StDone=FALSE OR _tStopTout.Q → ST_STOPPED

ST_STOPPED:
    CmdReset → ST_IDLE

ST_ABORTING:
    CmdAuto:=FALSE for all; wait _tStopTout.Q (no feedback wait) → ST_ABORTED

ST_ABORTED:
    CmdReset + IntlkEstop=TRUE → ST_IDLE; clear faults
```

After the CASE block, status booleans (StIdle, StStarting, StExecute, …, StAborted) are set by comparing State to each constant.

**Logic (SCL):**

// ── CmdAbort y E-Stop: max priority, for any state ────────────────────
IF (#Ctrl.Commands.CmdAbort AND NOT #xLastAbort)
    OR (NOT #Ctrl.Interlocks.IntlkEstop AND #Ctrl.Status.State <> #ST_ABORTED) THEN
    #Ctrl.Status.State := #ST_ABORTING;
    #Ctrl.Faults.FaultCode := USInt#2;   // EStop/Abort
    #Ctrl.Faults.FltAny := TRUE;
END_IF;

// ── PackML state machine ────────────────────────────────────────────────
CASE #Ctrl.Status.State OF
        
        // ── IDLE ─────────────────────────────────────────────────────────────────
    #ST_IDLE:
        #Ctrl.Status.StStep := 1;
        #Ctrl.Status.StDone := FALSE;
        #Ctrl.Faults.FltAny := FALSE;
        #Ctrl.Faults.FaultCode := USInt#0;
        
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled THEN
                #Devices[#n].Commands.CmdAuto := FALSE;
                #Devices[#n].Commands.CmdSP := 0.0;
            END_IF;
        END_FOR;
        
        IF #Ctrl.Commands.CmdStart AND NOT #xLastStart
            AND #Ctrl.Interlocks.IntlkReady
            AND #Ctrl.Interlocks.IntlkEstop THEN
            #Ctrl.Status.State := #ST_STARTING;
        END_IF;
        
        // ── STARTING ─────────────────────────────────────────────────────────────
    #ST_STARTING:
        // Verify all devices are enabled, without faults, ready to operate
        #xAllSafe := TRUE;
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled THEN
                IF #Devices[#n].Status.FltAny OR NOT #Devices[#n].Status.StReady THEN
                    #xAllSafe := FALSE;
                END_IF;
            END_IF;
        END_FOR;
        
        IF #xAllSafe THEN
            #Ctrl.Status.State := #ST_EXECUTE;
            #Ctrl.Status.StStep := 1;
        END_IF;
        
        IF NOT #Ctrl.Interlocks.IntlkEstop THEN
            #Ctrl.Status.State := #ST_ABORTING;
            #Ctrl.Faults.FaultCode := USInt#2;
            #Ctrl.Faults.FltAny := TRUE;
        END_IF;
        
        // ── EXECUTE ──────────────────────────────────────────────────────────────
    #ST_EXECUTE:
        // ── Device fault detection ──────────────────────────────
        #iFaultSeverity := USInt#0;
        #iFaultDevice := 0;
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled AND #Devices[#n].Status.FltAny THEN
                IF #Devices[#n].Config.FltSeverity > #iFaultSeverity THEN
                    #iFaultSeverity := #Devices[#n].Config.FltSeverity;
                    #iFaultDevice := #n;
                END_IF;
            END_IF;
        END_FOR;
        
        IF #iFaultSeverity = USInt#1 THEN
            // Hold — recoverable fault
            #Ctrl.Status.State := #ST_HOLDING;
            #Ctrl.Faults.FaultCode := USInt#3;
            #Ctrl.Faults.FaultDevice := #iFaultDevice;
            #Ctrl.Faults.FaultStep := #Ctrl.Status.StStep;
            #Ctrl.Faults.FltAny := TRUE;
            #xHoldFromPause := FALSE;
        ELSIF #iFaultSeverity = USInt#2 THEN
            // Abort — critical fault
            #Ctrl.Status.State := #ST_ABORTING;
            #Ctrl.Faults.FaultCode := USInt#3;
            #Ctrl.Faults.FaultDevice := #iFaultDevice;
            #Ctrl.Faults.FaultStep := #Ctrl.Status.StStep;
            #Ctrl.Faults.FltAny := TRUE;
        END_IF;
        
        // ── Step execution (only if state=EXECUTE) ───────────────────
        IF #Ctrl.Status.State = #ST_EXECUTE THEN
            
            CASE #Steps[#Ctrl.Status.StStep].StepType OF
                    
                    // StepType 0: Action
                0:
                    FOR #n := 1 TO #nDevices DO
                        IF #Devices[#n].Config.Enabled THEN
                            #dBitVal := SHR(IN := #Steps[#Ctrl.Status.StStep].ActMask,
                                            N := INT_TO_UINT(#n - 1)) AND DWORD#1;
                            IF #dBitVal = DWORD#1 THEN
                                #Devices[#n].Commands.CmdAuto :=
                                (SHR(IN := #Steps[#Ctrl.Status.StStep].ActTarget,
                                     N := INT_TO_UINT(#n - 1)) AND DWORD#1) = DWORD#1;
                                IF #Devices[#n].Config.DevType >= USInt#2 THEN
                                    #Devices[#n].Commands.CmdSP :=
                                    #Steps[#Ctrl.Status.StStep].ActSP[#n];
                                END_IF;
                            END_IF;
                        END_IF;
                    END_FOR;
                    
                    #xStepDone := TRUE;
                    IF #Steps[#Ctrl.Status.StStep].CondMask <> DWORD#0 THEN
                        FOR #n := 1 TO #nDevices DO
                            IF #Devices[#n].Config.Enabled THEN
                                #dBitVal := SHR(IN := #Steps[#Ctrl.Status.StStep].CondMask,
                                                N := INT_TO_UINT(#n - 1)) AND DWORD#1;
                                IF #dBitVal = DWORD#1 THEN
                                    IF (SHR(IN := #Steps[#Ctrl.Status.StStep].CondTarget,
                                                N := INT_TO_UINT(#n - 1)) AND DWORD#1) = DWORD#1 THEN
                                        #xStepDone := #xStepDone AND #Devices[#n].Status.StDone;
                                    ELSE
                                        #xStepDone := #xStepDone AND NOT #Devices[#n].Status.StDone;
                                    END_IF;
                                END_IF;
                            END_IF;
                        END_FOR;
                    END_IF;
                    
                    // StepType 1: Wait
                1:
                    #_tWait(IN := TRUE,
                            PT := DINT_TO_TIME(REAL_TO_DINT(#Steps[#Ctrl.Status.StStep].WaitTime * 1000.0)));
                    #xStepDone := #_tWait.Q;
                    
                    // StepType 2: External Condition
                2:
                    IF #Steps[#Ctrl.Status.StStep].ExtCondIdx >= 1 AND
                        #Steps[#Ctrl.Status.StStep].ExtCondIdx <= 8 THEN
                        #xStepDone := #ExtCond[#Steps[#Ctrl.Status.StStep].ExtCondIdx];
                    ELSE
                        #xStepDone := FALSE;
                    END_IF;
                    
                    // StepType 3: SubSequence
                3:
                    IF #Steps[#Ctrl.Status.StStep].SubSeqIdx >= 1 AND
                        #Steps[#Ctrl.Status.StStep].SubSeqIdx <= 8 THEN
                        #SubStart[#Steps[#Ctrl.Status.StStep].SubSeqIdx] := TRUE;
                        IF #Steps[#Ctrl.Status.StStep].Parallel THEN
                            #xStepDone := TRUE;
                        ELSE
                            #xStepDone := #SubDone[#Steps[#Ctrl.Status.StStep].SubSeqIdx];
                        END_IF;
                    ELSE
                        #xStepDone := FALSE;
                    END_IF;
                    
            END_CASE;
            
            // ── Step Timeout ──────────────────────────────────────────
            #_tStepTout(IN := (#Steps[#Ctrl.Status.StStep].Timeout > 0.0) AND NOT #xStepDone,
                       PT := DINT_TO_TIME(REAL_TO_DINT(#Steps[#Ctrl.Status.StStep].Timeout * 1000.0)));
            
            IF #_tStepTout.Q THEN
                #Ctrl.Status.State := #ST_ABORTING;
                #Ctrl.Faults.FaultCode := USInt#1;   // StepTimeout
                #Ctrl.Faults.FaultStep := #Ctrl.Status.StStep;
                #Ctrl.Faults.FltAny := TRUE;
            END_IF;
            
            // ── Next step ──────────────────────────────────────────────
            IF #xStepDone AND #Ctrl.Status.State = #ST_EXECUTE THEN
                IF #Steps[#Ctrl.Status.StStep].StepType = 3 THEN
                    #SubStart[#Steps[#Ctrl.Status.StStep].SubSeqIdx] := FALSE;
                END_IF;
                #_tWait(IN := FALSE,
                        PT := T#0MS);
                
                IF #Ctrl.Status.StStep < #nSteps THEN
                    #Ctrl.Status.StStep := #Ctrl.Status.StStep + 1;
                ELSE
                    #Ctrl.Status.State := #ST_COMPLETING;
                END_IF;
            END_IF;
            
        END_IF;
        
        // Cmd Pause / Stop handle
        IF #Ctrl.Commands.CmdPause AND NOT #xLastPause THEN
            #Ctrl.Status.State := #ST_PAUSING;
            #xHoldFromPause := TRUE;
        END_IF;
        IF #Ctrl.Commands.CmdStop AND NOT #xLastStop THEN
            #Ctrl.Status.State := #ST_STOPPING;
        END_IF;
        
        // ── COMPLETING ───────────────────────────────────────────────────────────
    #ST_COMPLETING:
        // Controlled Shutdown: default state cmd to all devices
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled THEN
                #Devices[#n].Commands.CmdAuto := FALSE;
                #Devices[#n].Commands.CmdSP := 0.0;
            END_IF;
        END_FOR;
        
        // Verify default state
        #xAllSafe := TRUE;
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled AND #Devices[#n].Status.StDone THEN
                #xAllSafe := FALSE;
            END_IF;
        END_FOR;
        
        IF #xAllSafe THEN
            #Ctrl.Status.State := #ST_COMPLETE;
            #Ctrl.Status.StDone := TRUE;
        END_IF;
        
        // ── COMPLETE ─────────────────────────────────────────────────────────────
    #ST_COMPLETE:
        IF #Ctrl.Commands.CmdReset AND NOT #xLastReset THEN
            #Ctrl.Status.State := #ST_IDLE;
        END_IF;
        
        // ── HOLDING ──────────────────────────────────────────────────────────────
    #ST_HOLDING:
        // Safe-state confirmation
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled THEN
                #Devices[#n].Commands.CmdAuto := FALSE;
                #Devices[#n].Commands.CmdSP := 0.0;
            END_IF;
        END_FOR;
        
        #xAllSafe := TRUE;
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled AND #Devices[#n].Status.StDone THEN
                #xAllSafe := FALSE;
            END_IF;
        END_FOR;
        
        // Transition timeout HOLDING → ABORTING
        #_tHoldTout(
                    IN := (#Ctrl.Config.HoldTimeout > 0.0),
                    PT := DINT_TO_TIME(REAL_TO_DINT(#Ctrl.Config.HoldTimeout * 1000.0)));
        IF #_tHoldTout.Q THEN
            #Ctrl.Status.State := #ST_ABORTING;
            #Ctrl.Faults.FaultCode := USInt#4;   // HoldTimeout
            #_tHoldTout(IN := FALSE,
                        PT := T#0MS);
        END_IF;
        
        IF #xAllSafe THEN
            #Ctrl.Status.State := #ST_HELD;
            #_tHoldTout(IN := FALSE,
                        PT := T#0MS);
        END_IF;
        
        // ── HELD ─────────────────────────────────────────────────────────────────
    #ST_HELD:
        // Manual resume
        IF #Ctrl.Commands.CmdResume AND NOT #xLastResume THEN
            #Ctrl.Status.State := #ST_RESUMING;
        END_IF;
        
        // Auto-resume: faults cleared and Config.AutoResume = TRUE
        IF #Ctrl.Config.AutoResume THEN
            #xAllSafe := TRUE;
            FOR #n := 1 TO #nDevices DO
                IF #Devices[#n].Config.Enabled AND #Devices[#n].Status.FltAny THEN
                    #xAllSafe := FALSE;
                END_IF;
            END_FOR;
            
            #_tAutoResume(
                          IN := #xAllSafe AND (#Ctrl.Config.AutoResumeTime > 0.0),
                          PT := DINT_TO_TIME(REAL_TO_DINT(#Ctrl.Config.AutoResumeTime * 1000.0)));
            
            IF #xAllSafe AND (#Ctrl.Config.AutoResumeTime = 0.0 OR #_tAutoResume.Q) THEN
                #Ctrl.Status.State := #ST_RESUMING;
                #Ctrl.Faults.FltAny := FALSE;
                #Ctrl.Faults.FaultCode := USInt#0;
                #_tAutoResume(IN := FALSE,
                              PT := T#0MS);
            END_IF;
        END_IF;
        
        IF #Ctrl.Commands.CmdStop AND NOT #xLastStop THEN
            #Ctrl.Status.State := #ST_STOPPING;
        END_IF;
        
        // ── RESUMING ─────────────────────────────────────────────────────────────
    #ST_RESUMING:
        // Verify conditions before resuming
        #xAllSafe := TRUE;
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled THEN
                IF #Devices[#n].Status.FltAny OR NOT #Devices[#n].Status.StReady THEN
                    #xAllSafe := FALSE;
                END_IF;
            END_IF;
        END_FOR;
        
        IF #xAllSafe THEN
            #Ctrl.Status.State := #ST_EXECUTE;
            #Ctrl.Faults.FltAny := FALSE;
            #Ctrl.Faults.FaultCode := USInt#0;
        END_IF;
        
        // ── PAUSING ──────────────────────────────────────────────────────────────
    #ST_PAUSING:
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled THEN
                #Devices[#n].Commands.CmdAuto := FALSE;
                #Devices[#n].Commands.CmdSP := 0.0;
            END_IF;
        END_FOR;
        
        #xAllSafe := TRUE;
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled AND #Devices[#n].Status.StDone THEN
                #xAllSafe := FALSE;
            END_IF;
        END_FOR;
        
        IF #xAllSafe THEN
            #Ctrl.Status.State := #ST_PAUSED;
        END_IF;
        
        // ── PAUSED ───────────────────────────────────────────────────────────────
    #ST_PAUSED:
        IF #Ctrl.Commands.CmdResume AND NOT #xLastResume THEN
            #Ctrl.Status.State := #ST_RESUMING;
        END_IF;
        IF #Ctrl.Commands.CmdStop AND NOT #xLastStop THEN
            #Ctrl.Status.State := #ST_STOPPING;
        END_IF;
        
        // ── STOPPING ─────────────────────────────────────────────────────────────
    #ST_STOPPING:
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled THEN
                #Devices[#n].Commands.CmdAuto := FALSE;
                #Devices[#n].Commands.CmdSP := 0.0;
            END_IF;
        END_FOR;
        
        #xAllSafe := TRUE;
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled AND #Devices[#n].Status.StDone THEN
                #xAllSafe := FALSE;
            END_IF;
        END_FOR;
        
        #_tStopTout(
                    IN := NOT #xAllSafe,
                    PT := DINT_TO_TIME(REAL_TO_DINT(#Ctrl.Config.StopTimeout * 1000.0)));
        
        IF #xAllSafe OR #_tStopTout.Q THEN
            #Ctrl.Status.State := #ST_STOPPED;
            #_tStopTout(IN := FALSE,
                        PT := T#0MS);
        END_IF;
        
        // ── STOPPED ──────────────────────────────────────────────────────────────
    #ST_STOPPED:
        IF #Ctrl.Commands.CmdReset AND NOT #xLastReset THEN
            #Ctrl.Status.State := #ST_IDLE;
        END_IF;
        
        // ── ABORTING ─────────────────────────────────────────────────────────────
    #ST_ABORTING:
        // Emergency Shutdown — agressive, does not wait for default position confirmation
        FOR #n := 1 TO #nDevices DO
            IF #Devices[#n].Config.Enabled THEN
                #Devices[#n].Commands.CmdAuto := FALSE;
                #Devices[#n].Commands.CmdSP := 0.0;
            END_IF;
        END_FOR;
        
        #_tStopTout(
                    IN := TRUE,
                    PT := DINT_TO_TIME(REAL_TO_DINT(#Ctrl.Config.StopTimeout * 1000.0)));
        
        // ABORTING only waiting for timeout
        IF #_tStopTout.Q THEN
            #Ctrl.Status.State := #ST_ABORTED;
            #_tStopTout(IN := FALSE,
                        PT := T#0MS);
        END_IF;
        
        // ── ABORTED ──────────────────────────────────────────────────────────────
    #ST_ABORTED:
        // Reset only if E-Stop OK and reset cmd
        IF #Ctrl.Commands.CmdReset AND NOT #xLastReset
            AND #Ctrl.Interlocks.IntlkEstop THEN
            #Ctrl.Status.State := #ST_IDLE;
            #Ctrl.Faults.FltAny := FALSE;
            #Ctrl.Faults.FaultCode := USInt#0;
        END_IF;
        
END_CASE;

// ── Status outputs ────────────────────────────────────────────────────────────
#Ctrl.Status.StIdle := (#Ctrl.Status.State = #ST_IDLE);
#Ctrl.Status.StStarting := (#Ctrl.Status.State = #ST_STARTING);
#Ctrl.Status.StExecute := (#Ctrl.Status.State = #ST_EXECUTE);
#Ctrl.Status.StCompleting := (#Ctrl.Status.State = #ST_COMPLETING);
#Ctrl.Status.StComplete := (#Ctrl.Status.State = #ST_COMPLETE);
#Ctrl.Status.StHolding := (#Ctrl.Status.State = #ST_HOLDING);
#Ctrl.Status.StHeld := (#Ctrl.Status.State = #ST_HELD);
#Ctrl.Status.StResuming := (#Ctrl.Status.State = #ST_RESUMING);
#Ctrl.Status.StPausing := (#Ctrl.Status.State = #ST_PAUSING);
#Ctrl.Status.StPaused := (#Ctrl.Status.State = #ST_PAUSED);
#Ctrl.Status.StStopping := (#Ctrl.Status.State = #ST_STOPPING);
#Ctrl.Status.StStopped := (#Ctrl.Status.State = #ST_STOPPED);
#Ctrl.Status.StAborting := (#Ctrl.Status.State = #ST_ABORTING);
#Ctrl.Status.StAborted := (#Ctrl.Status.State = #ST_ABORTED);

// ── Edge memories ────────────────────────────────────────────────────────
#xLastStart := #Ctrl.Commands.CmdStart;
#xLastStop := #Ctrl.Commands.CmdStop;
#xLastPause := #Ctrl.Commands.CmdPause;
#xLastResume := #Ctrl.Commands.CmdResume;
#xLastAbort := #Ctrl.Commands.CmdAbort;
#xLastReset := #Ctrl.Commands.CmdReset;


**T-STD-001 adaptations:**
- `nDevices` := actual valve count (1..15). Runner iterates 1..nDevices and skips Enabled=FALSE.
- `Devices` := `DB_DEVICES.Valves` (ARRAY[1..32]; positions 16..32 have Enabled=FALSE).
- `ExtCond`, `SubDone`, `SubStart`, `SubHeld`, `SubPaused` are declared as OB1 TEMP arrays (stubs — not connected to external logic in T-STD-001 standalone topology).

### 7.2 FB_SeqMaster (FB310) — PackML Multi-Runner Master

Language: **SCL**. Documented here for completeness. **Not called in T-STD-001 OB1** — T-STD-001 uses the standalone Runner topology (one FB_SeqRunner, no master).

**Interface:**
```
FB_SeqMaster (SCL)                                     // FB310
  IN:
    nSubSeqs      : Int                              // Number of active sub-sequences (1..8)
  IN_OUT:
    Ctrl          : UDT_MasterCtrl
    SubCtrl       : ARRAY[1..8] OF UDT_SeqCtrl      // Each sub-runner's Ctrl
  STATIC:
    xAllDone      : Bool
    xAllSafe      : Bool
    xLastStart    : Bool
    xLastStop     : Bool
    xLastPause    : Bool
    xLastResume   : Bool
    xLastAbort    : Bool
    xLastReset    : Bool
  TEMP:
    n             : Int
    iFaultSub     : USInt
    iPolicy       : USInt
```

**PackML state constants:**

| State | Value |
|---|---|
| ST_IDLE | 0 |
| ST_STARTING | 1 |
| ST_EXECUTE | 2 |
| ST_COMPLETING | 3 |
| ST_COMPLETE | 4 |
| ST_HOLDING | 5 |
| ST_HELD | 6 |
| ST_RESUMING | 7 |
| ST_PAUSING | 8 |
| ST_PAUSED | 9 |
| ST_STOPPING | 11 |
| ST_STOPPED | 12 |
| ST_ABORTING | 13 |
| ST_ABORTED | 14 |

**Purpose:** Coordinates multiple FB_SeqRunner instances. Handles global E-Stop propagation, sub-sequence fault policy (AbortAll/AbortOnly/HoldAll/HoldOnly per UDT_MasterSubConfig), and cross-runner start/stop/pause/resume commands. Include when a project requires two or more concurrent or mutually exclusive sub-sequences (T-STD-002+).

**Logic (SCL):**

// ── CmdAbort y E-Stop: max priority, for any state ────────────────────
IF (#Ctrl.Commands.CmdAbort AND NOT #xLastAbort)
    OR NOT #Ctrl.Interlocks.IntlkEstop THEN
    #Ctrl.Status.State := #ST_ABORTING;
    FOR #n := 1 TO #nSubSeqs DO
        #SubCtrl[#n].Commands.CmdAbort := TRUE;
    END_FOR;
END_IF;

// ── Sub-sequence fault detection ─────────────────────────────────────────
FOR #n := 1 TO #nSubSeqs DO
    IF #SubCtrl[#n].Status.StAborted OR #SubCtrl[#n].Status.StHeld THEN
        #iFaultSub := INT_TO_USINT(#n);
        
        IF #Ctrl.Config.SubPolicy[#n].IsMandatory THEN
            #iPolicy := #Ctrl.Config.SubPolicy[#n].OnFaultPolicy;
            
            CASE #iPolicy OF
                0:  // AbortAll
                    #Ctrl.Status.State := #ST_ABORTING;
                    #Ctrl.Faults.FaultSubSeq := #iFaultSub;
                    #Ctrl.Faults.FltAny := TRUE;
                    FOR #n := 1 TO #nSubSeqs DO
                        #SubCtrl[#n].Commands.CmdAbort := TRUE;
                    END_FOR;
                1:  // AbortOnly — Only faulted runner
                    #Ctrl.Faults.FaultSubSeq := #iFaultSub;
                    #Ctrl.Faults.FltAny := TRUE;
                2:  // HoldAll
                    #Ctrl.Status.State := #ST_HELD;
                    #Ctrl.Faults.FaultSubSeq := #iFaultSub;
                    #Ctrl.Faults.FltAny := TRUE;
                    FOR #n := 1 TO #nSubSeqs DO
                        #SubCtrl[#n].Commands.CmdStop := TRUE;
                    END_FOR;
                3:  // HoldOnly — Only faulted runner
                    #Ctrl.Faults.FaultSubSeq := #iFaultSub;
                    #Ctrl.Faults.FltAny := TRUE;
            END_CASE;
        END_IF;
    END_IF;
END_FOR;

// ── PackML Master state machine ────────────────────────────────────────────────
CASE #Ctrl.Status.State OF
        
    #ST_IDLE:
        IF #Ctrl.Commands.CmdStart AND NOT #xLastStart
            AND #Ctrl.Interlocks.IntlkReady
            AND #Ctrl.Interlocks.IntlkEstop THEN
            #Ctrl.Status.State := #ST_EXECUTE;
            #Ctrl.Status.StDone := FALSE;
            // Sub-sequences launch
            FOR #n := 1 TO #nSubSeqs DO
                #SubCtrl[#n].Commands.CmdStart := TRUE;
                #SubCtrl[#n].Interlocks.IntlkReady := TRUE;
                #SubCtrl[#n].Interlocks.IntlkEstop := #Ctrl.Interlocks.IntlkEstop;
            END_FOR;
        END_IF;
        
    #ST_EXECUTE:
        // E-Stop propagation
        FOR #n := 1 TO #nSubSeqs DO
            #SubCtrl[#n].Interlocks.IntlkEstop := #Ctrl.Interlocks.IntlkEstop;
        END_FOR;
        
        // Verify sub-sequences completion
        #xAllDone := TRUE;
        FOR #n := 1 TO #nSubSeqs DO
            IF NOT #SubCtrl[#n].Status.StComplete
                AND NOT #SubCtrl[#n].Status.StIdle THEN
                #xAllDone := FALSE;
            END_IF;
        END_FOR;
        IF #xAllDone THEN
            #Ctrl.Status.State := #ST_IDLE;
            #Ctrl.Status.StDone := TRUE;
        END_IF;
        
        IF #Ctrl.Commands.CmdPause AND NOT #xLastPause THEN
            #Ctrl.Status.State := #ST_PAUSED;
            FOR #n := 1 TO #nSubSeqs DO
                #SubCtrl[#n].Commands.CmdPause := TRUE;
            END_FOR;
        END_IF;
        
        IF #Ctrl.Commands.CmdStop AND NOT #xLastStop THEN
            #Ctrl.Status.State := #ST_STOPPING;
            FOR #n := 1 TO #nSubSeqs DO
                #SubCtrl[#n].Commands.CmdStop := TRUE;
            END_FOR;
        END_IF;
        
    #ST_HELD:
        IF #Ctrl.Commands.CmdResume AND NOT #xLastResume THEN
            #Ctrl.Status.State := #ST_EXECUTE;
            FOR #n := 1 TO #nSubSeqs DO
                #SubCtrl[#n].Commands.CmdResume := TRUE;
            END_FOR;
            #Ctrl.Faults.FltAny := FALSE;
            #Ctrl.Faults.FaultCode := USInt#0;
        END_IF;
        
    #ST_PAUSED:
        IF #Ctrl.Commands.CmdResume AND NOT #xLastResume THEN
            #Ctrl.Status.State := #ST_EXECUTE;
            FOR #n := 1 TO #nSubSeqs DO
                #SubCtrl[#n].Commands.CmdResume := TRUE;
            END_FOR;
        END_IF;
        IF #Ctrl.Commands.CmdStop AND NOT #xLastStop THEN
            #Ctrl.Status.State := #ST_STOPPING;
            FOR #n := 1 TO #nSubSeqs DO
                #SubCtrl[#n].Commands.CmdStop := TRUE;
            END_FOR;
        END_IF;
        
    #ST_STOPPING:
        #xAllSafe := TRUE;
        FOR #n := 1 TO #nSubSeqs DO
            IF NOT #SubCtrl[#n].Status.StStopped
                AND NOT #SubCtrl[#n].Status.StIdle THEN
                #xAllSafe := FALSE;
            END_IF;
        END_FOR;
        IF #xAllSafe THEN
            #Ctrl.Status.State := #ST_STOPPED;
        END_IF;
        
    #ST_STOPPED:
        IF #Ctrl.Commands.CmdReset AND NOT #xLastReset THEN
            #Ctrl.Status.State := #ST_IDLE;
            FOR #n := 1 TO #nSubSeqs DO
                #SubCtrl[#n].Commands.CmdReset := TRUE;
            END_FOR;
        END_IF;
        
    #ST_ABORTING:
        #xAllSafe := TRUE;
        FOR #n := 1 TO #nSubSeqs DO
            IF NOT #SubCtrl[#n].Status.StAborted
                AND NOT #SubCtrl[#n].Status.StIdle THEN
                #xAllSafe := FALSE;
            END_IF;
        END_FOR;
        IF #xAllSafe THEN
            #Ctrl.Status.State := #ST_ABORTED;
        END_IF;
        
    #ST_ABORTED:
        IF #Ctrl.Commands.CmdReset AND NOT #xLastReset
            AND #Ctrl.Interlocks.IntlkEstop THEN
            #Ctrl.Status.State := #ST_IDLE;
            #Ctrl.Faults.FltAny := FALSE;
            #Ctrl.Faults.FaultCode := USInt#0;
            FOR #n := 1 TO #nSubSeqs DO
                #SubCtrl[#n].Commands.CmdReset := TRUE;
            END_FOR;
        END_IF;
        
END_CASE;

// ── Status outputs ─────────────────────────────────────────────────────────────
#Ctrl.Status.StIdle := (#Ctrl.Status.State = #ST_IDLE);
#Ctrl.Status.StExecute := (#Ctrl.Status.State = #ST_EXECUTE);
#Ctrl.Status.StHeld := (#Ctrl.Status.State = #ST_HELD);
#Ctrl.Status.StPaused := (#Ctrl.Status.State = #ST_PAUSED);
#Ctrl.Status.StStopped := (#Ctrl.Status.State = #ST_STOPPED);
#Ctrl.Status.StAborted := (#Ctrl.Status.State = #ST_ABORTED);

// ── Edge memories ─────────────────────────────────────────────────────────
#xLastStart := #Ctrl.Commands.CmdStart;
#xLastStop := #Ctrl.Commands.CmdStop;
#xLastPause := #Ctrl.Commands.CmdPause;
#xLastResume := #Ctrl.Commands.CmdResume;
#xLastAbort := #Ctrl.Commands.CmdAbort;
#xLastReset := #Ctrl.Commands.CmdReset;


---

## 8. Common Module Designs

### 8.1 FB_DI — Digital Input (T-MOD-024)

Used for: SYS_AIR_OK, operator pushbuttons, E-stop monitoring signal. NOT for valve position feedback (handled inside FB_VALV_OnOff).

**UDT_DI:**
```
UDT_DI:
  State        : BOOL    // Debounced, polarity-corrected state
  RisingEdge   : BOOL    // One-scan pulse on FALSE→TRUE
  FallingEdge  : BOOL    // One-scan pulse on TRUE→FALSE
```

**Interface:**
```
FB_DI                                                  // FB900
  IN:
    In           : BOOL    // Raw physical DI bit
    Invert       : BOOL    // TRUE = active-low (NC contact)
    Debounce_ms  : INT     // Debounce time in ms (default: 10)
  IN_OUT:
    Out          : UDT_DI  // Debounced signal
  Static:
    tDebounce    : IEC_TIMER
    xLastState   : BOOL
```

**Logic (LAD):**
TON debounce on raw input change → apply inversion → one-scan edge detection using `_lastState`.

### 8.2 FB_DO — Digital Output (T-MOD-025)

Used for: auxiliary outputs (air preparation enable, warning beacon). NOT for valve solenoids (written by FB_VALV_OnOff).

**UDT_DO:**
```
UDT_DO:
  State        : BOOL    // Commanded state
  FbkMismatch  : BOOL    // Output commanded but feedback disagrees (wiring fault)
```

**Interface:**
```
FB_DO                                                  // FB910
  IN:
    Cmd          : BOOL    // Command from logic
    Fbk          : BOOL    // Physical readback (wire to Cmd if not available)
    FbkEnabled   : BOOL    // Enable mismatch detection
    FbkTout_ms   : INT     // Time before mismatch fault (default: 200ms)
  IN_OUT:
    Out : UDT_DO
  OUT:
    Out      : BOOL    // Drive this to the physical %Q tag
  Static:
    tFbkTout     : IEC_TIMER
```

**Logic (LAD):**
Copy Cmd to PhysOut and Out.State. If FbkEnabled: start TON when Cmd≠Fbk; if TON.Q, set FbkMismatch.

### 8.3 FB_ESTOP — E-Stop Chain (T-MOD-008)

For PLc on a valve cluster (Cat. 2 with safety relay, no F-CPU). The safety relay handles the actual PLc safety function in hardware. FB_ESTOP monitors the relay output status and manages the PLC-level "permission to run" signal.

**UDT_EStopCtrl:**
```
UDT_EStopCtrl:
  StOK          : BOOL    // Chain is safe — TRUE = operate allowed
  StTripped     : BOOL    // E-stop is activated (button pressed or wire break)
  StResetReq    : BOOL    // Chain restored, awaiting operator reset
  FltChain      : BOOL    // Monitored channel disagrees (reserved for Cat. 3+ upgrade)
```

**Interface:**
```
FB_ESTOP                                               // FB400
  IN:
    SftyRelayOut : BOOL    // Safety relay monitored output — TRUE = relay energized = safe
    CmdReset     : BOOL    // Reset request (rising edge, physical button or HMI)
  IN_OUT:
    Ctrl         : UDT_EStopCtrl
  Static:
    xLastRelay   : BOOL
    xLastReset   : BOOL
```

**Logic (LAD):**
```
Falling edge SftyRelayOut: Ctrl.StOK := FALSE; Ctrl.StTripped := TRUE
Rising edge SftyRelayOut:  Ctrl.StTripped := FALSE; Ctrl.StResetReqd := TRUE
Rising edge CmdReset (when StResetReqd): Ctrl.StOK := TRUE; Ctrl.StResetReqd := FALSE
```

`DB_SYS.EStop.StOK` feeds `DB_SEQUENCES.RunnerCtrl.Interlocks.IntlkEstop` in OB1 (wired
inline before calling FB_SeqRunner — see Section 10.2).

### 8.4 FB_MOTOR — Motor Control, DOL (T-MOD-001)

Standard reusable module for direct-on-line motors (pumps, compressors, fans). Always present;
always receives `Dev : UDT_DeviceCtrl`. The sequence master or a services sub-sequence commands
the motor via `Dev.Commands.CmdAuto`. In T-STD-001, if no sub-sequence is defined for the motor,
`Dev.Commands.CmdAuto` remains FALSE and the motor operates in manual mode only.

When no pump motor is present in the project: set `DB_SYS.Motor.Enabled := FALSE` and
`DB_DEVICES.Motors[1].Config.Enabled := FALSE` in DB initial values — do NOT omit the block.

**Interface:**
```
FB_MOTOR                                               // FB100
  IN_OUT:
    IO                 : UDT_MotorIO     // Physical I/O — pass DB_MOTOR_IO.IO
    Ctrl               : UDT_MotorCtrl   // Control and status — pass DB_SYS.Motor
    Dev                : UDT_DeviceCtrl  // Runner interface — pass DB_DEVICES.Motors[1]
  Static:
    tTout              : IEC_TIMER    // Run confirmation timeout
    tSimRun            : IEC_TIMER    // Sim run feedback timer
    tRunSecTon         : IEC_TIMER    // 1 s auto-reset pulse for run-hours accumulation
    xLastReset         : BOOL
    xLastEffectiveRun  : BOOL         // Last EffectiveRun state for activation counter edge detection
    xLastResetCnt      : BOOL         // Last CmdResetCounters state for edge detection
  TEMP:
    xEffectiveRun      : BOOL
    xEffectiveFbkRun   : BOOL
```

**Logic (LAD):**

**Config sync (before Step 0):**
```scl
Dev.Config.DevType     := USInt#1;         // Motor
Dev.Config.Enabled     := Ctrl.Config.Enabled;
Dev.Config.FltSeverity := Ctrl.Config.FltSeverity;
```

**Step 0 — Disabled check:**
```
IF NOT Ctrl.Config.Enabled THEN
    IO.Outputs.CmdRun  := FALSE
    Ctrl.Status.State := 4
    // Skip remaining steps
END_IF
```

**Step 1 — Mode resolution:**
```
IF Ctrl.Commands.CmdManual THEN
    xEffectiveRun := Ctrl.Commands.CmdManRun
ELSE
    xEffectiveRun := Dev.Commands.CmdAuto   // Runner (or sub-sequence) writes this
END_IF
```

**Step 2 — SimMode feedback generation:**
```
tSimRun.PT := Ctrl.Config.SimDelay * 1000   //Conversion is automatic with MUL

IF Ctrl.Config.SimMode THEN
    tSimRun.IN := xEffectiveRun

    xEffectiveFbkRun := tSimRun.Q
ELSE
    xEffectiveFbkRun := IO.Inputs.FbkRunning
END_IF
```

Physical output (IO.Outputs.CmdRun) is still written to hardware in SimMode — this allows verification during FAT. Only the feedback path is intercepted.

**Step 3 — Output to field (fault locks output to safe state):**
```
IO.Outputs.CmdRun := xEffectiveRun AND NOT Ctrl.Faults.FltAny
```

**Step 4 — Run confirmation timeout:**
```
tTout.PT := Ctrl.Config.ToutRun * 1000   //Conversion is automatic with MUL
tTout.IN := xEffectiveRun AND NOT xEffectiveFbkRun

IF tTout.Q THEN
    Ctrl.Faults.FltTimeout := TRUE     //SET
END_IF
```

**Step 5 — Overload fault (NC-wired thermal relay: FALSE = tripped):**
```
Ctrl.Faults.FltOverload := NOT IO.Inputs.FbkOverload
```

**Step 6 — Status:**
```
Ctrl.Status.StRunning := xEffectiveFbkRun AND xEffectiveRun
Ctrl.Status.StStopped := NOT xEffectiveFbkRun
```

**Step 7 — Fault aggregation and reset:**
```
Ctrl.Faults.FltAny := Ctrl.Faults.FltTimeout OR Ctrl.Faults.FltOverload

IF Ctrl.Commands.CmdReset AND NOT xLastReset THEN
    Ctrl.Faults.FltTimeout  := FALSE
    Ctrl.Faults.FltOverload := FALSE
    xEffectiveRun    := FALSE
END_IF
xLastReset := Ctrl.CmdReset
```

**Step 8 — State calculation:**
``` SCL justified
IF NOT Ctrl.Config.Enabled THEN
    Ctrl.State := 4
ELSIF Ctrl.FltAny THEN
    Ctrl.State := 3
ELSIF Ctrl.StRunning THEN
    Ctrl.State := 2
ELSIF NOT EffectiveRun AND Ctrl.StStopped AND NOT Ctrl.ModeManual THEN
    Ctrl.State := 1
ELSE
    Ctrl.State := 0
END_IF
```

**Step 9 — StReady and Dev.Status sync:**
```scl
Ctrl.StReady := Ctrl.Enabled AND NOT Ctrl.FltAny AND Ctrl.StStopped;

Dev.Status.StDone  := Ctrl.StRunning;    // Active state = Running
Dev.Status.StReady := Ctrl.StReady;
Dev.Status.FltAny  := Ctrl.FltAny;
```

**Step 10 — Activation and run-hours counters:**
```scl
// Rising edge EffectiveRun → increment start counter (clamped to prevent UDINT overflow)
IF EffectiveRun AND NOT _lastEffRun THEN
    IF Ctrl.ActivationCount < 4294967295 THEN
        Ctrl.ActivationCount := Ctrl.ActivationCount + 1;
    END_IF;
END_IF;
_lastEffRun := EffectiveRun;

// TON auto-reset: pulses every 1 s while confirmed running
// LAD: _tRunSecTon.IN driven by (Ctrl.StRunning AND NOT _tRunSecTon.Q)
_tRunSecTon.IN := Ctrl.StRunning AND NOT _tRunSecTon.Q;
_tRunSecTon.PT := T#1S;
IF _tRunSecTon.Q THEN
    IF Ctrl.RunHours < 999999.9 THEN
        Ctrl.RunHours := Ctrl.RunHours + 0.000277778;    // 1 s = 1/3600 h
    END_IF;
END_IF;

// Rising edge CmdResetCounters → clear both counters
IF Ctrl.CmdResetCounters AND NOT _lastRstCnt THEN
    Ctrl.ActivationCount := 0;
    Ctrl.RunHours        := 0.0;
END_IF;
_lastRstCnt := Ctrl.CmdResetCounters;
```

### 8.5 FB_TIMER_EXT — Timer Extended (T-MOD-031)

**UDT_TimerExt:**
```
UDT_TimerExt:
  Running      : BOOL    // Timer is active and counting
  Done         : BOOL    // Timer has elapsed (stays TRUE until reset)
  Elapsed_ms   : DINT    // Milliseconds elapsed since last start
  PT_ms        : DINT    // Current preset value in ms
```

**Interface:**
```
FB_TIMER_EXT                                           // FB1000
  IN:
    Enable       : BOOL    // Start/enable (like TON.IN)
    PT_ms        : DINT    // Preset time in ms
    AutoReset    : BOOL    // TRUE = auto-resets on Done (repeating pulse)
  IN_OUT:
    T : UDT_TimerExt
  Static:
    _ton         : TON
    _lastEnable  : BOOL
```

Note: FB_VALV_OnOff uses native TON internally. FB_SeqRunner uses native TON for all its timers
(_tStepTout, _tStopTout, etc.). FB_TIMER_EXT is available as a utility module for any application
block needing DINT-based preset or elapsed time readback. Language: LAD.

### 8.6 FB_AlarmHandler and FB_AlarmManager (T-MOD-018a / T-MOD-018b)

Two-layer alarm architecture: **FB_AlarmHandler** (one static instance per alarm, O(1) access)
+ **FB_AlarmManager** (wrapper FB that calls all 37 instances and processes global commands).

Language: **SCL** — justified for both: FB_AlarmHandler uses conditional history buffer writes
with index arithmetic; FB_AlarmManager iterates all alarm states via FOR loops.

#### FB_AlarmHandler (FB700)

One instance per alarm. Manages the full lifecycle: active/inactive edges, history write on
activation, TimeCleared update on deactivation, individual ACK.

```
FB_AlarmHandler (SCL)                                  // FB700
  IN:
    IsActive      : Bool     // Alarm source signal
    AlarmID       : DInt     // Encoded alarm ID (see Section 3.10)
    Category      : USInt    // 1=Valves 2=System
    Severity      : USInt    // 0=Info 1=Warning 2=Error 3=Critical
    Device        : USInt    // Valve index 1..15; 0=system
    CmdAck        : Bool     // Individual alarm ACK (not used in T-STD-001; pass FALSE)
    EnableHistory : Bool     // FALSE = skip history write (saves cycle time)
  IN_OUT:
    State         : UDT_AlarmState       // Pass DB_ALARMS.Valves[n] or System[n]
    HistBuf       : UDT_AlarmHistBuffer  // Pass DB_HIST.Buf
  Static:
    _PrevActive   : Bool
    _PrevAck      : Bool
    _HistIdx      : Int      // Index of open history entry for this alarm
    _InHistory    : Bool     // TRUE = open entry exists in history
```

Logic (per scan):
1. **Rising edge IsActive:** `State.Active:=TRUE; State.Count+1; State.AlarmID/Severity/Category/Device set; State.TimeRaised=RD_SYS_T; State.Acknowledged:=FALSE`. If EnableHistory: write to `HistBuf.Entries[Head]`, save `_HistIdx:=Head`, advance Head (modulo MaxSize), update Count/Full.
2. **Falling edge IsActive:** `State.Active:=FALSE; State.TimeCleared=RD_SYS_T`. If EnableHistory AND _InHistory: `HistBuf.Entries[_HistIdx].TimeCleared:=sysTime; _InHistory:=FALSE`.
3. **Rising edge CmdAck:** `State.Acknowledged:=TRUE`. If EnableHistory AND _InHistory: `HistBuf.Entries[_HistIdx].Acknowledged:=TRUE`.
4. Update `_PrevActive` and `_PrevAck`.

#### FB_AlarmManager (FB710)

Declared as **FB** (not FC) to have VAR statics for edge detection — eliminates the `%M0.0`/`%M0.1`
bug present in the reference `FC_AlarmManager`. One call in OB1 replaces 37 individual calls.

```
FB_AlarmManager (SCL)                                  // FB710
  IN:
    CmdAckAll     : Bool     // ACK all alarms (pass DB_SYS.AlarmCmdAckAll)
    CmdClear      : Bool     // Clear history buffer (pass DB_SYS.AlarmCmdClear)
    EnableHistory : Bool     // Pass TRUE always in T-STD-001
  Static:
    _prevAckAll   : Bool     // Edge detection for CmdAckAll — replaces %M0.0
    _prevClear    : Bool     // Edge detection for CmdClear — replaces %M0.1
    // 37 multi-instance FB_AlarmHandler (no separate iDBs — all share iDB_AlarmManager):
    // Valve timeout (15 instances):
    _hV01_Tout .. _hV15_Tout : FB_AlarmHandler
    // Valve discrepancy (15 instances):
    _hV01_Disc .. _hV15_Disc : FB_AlarmHandler
    // System (7 instances):
    _hSysEstop, _hSysAir, _hSysHold, _hSysStepTout,
    _hSysSeqStart, _hSysSeqStop, _hSysWdg : FB_AlarmHandler
```

**Body — valve calls (pattern for n=1..15):**
```scl
// Valve n: timeout = Valves[(2n-2)], discrepancy = Valves[(2n-1)]
// AlarmID: timeout = 110000+(2n-1), discrepancy = 110000+2n
// Example shown for n=1 and n=2; follow pattern for n=3..15

_hV01_Tout(IsActive := DB_VALV_CTRL.CTRL[1].FltTimeout,
            AlarmID := DInt#110001, Category := USInt#1, Severity := USInt#3,
            Device := USInt#1, CmdAck := FALSE, EnableHistory := EnableHistory,
            State := DB_ALARMS.Valves[0], HistBuf := DB_HIST.Buf);

_hV01_Disc(IsActive := DB_VALV_CTRL.CTRL[1].FltDiscrepancy,
            AlarmID := DInt#110002, Category := USInt#1, Severity := USInt#3,
            Device := USInt#1, CmdAck := FALSE, EnableHistory := EnableHistory,
            State := DB_ALARMS.Valves[1], HistBuf := DB_HIST.Buf);

_hV02_Tout(IsActive := DB_VALV_CTRL.CTRL[2].FltTimeout,
            AlarmID := DInt#110003, Category := USInt#1, Severity := USInt#3,
            Device := USInt#2, CmdAck := FALSE, EnableHistory := EnableHistory,
            State := DB_ALARMS.Valves[2], HistBuf := DB_HIST.Buf);

_hV02_Disc(IsActive := DB_VALV_CTRL.CTRL[2].FltDiscrepancy,
            AlarmID := DInt#110004, Category := USInt#1, Severity := USInt#3,
            Device := USInt#2, CmdAck := FALSE, EnableHistory := EnableHistory,
            State := DB_ALARMS.Valves[3], HistBuf := DB_HIST.Buf);

// ... pattern continues: _hVnn_Tout/Disc for n=3..15
// AlarmID for Vn: timeout = DInt#(110000 + 2n - 1), disc = DInt#(110000 + 2n)
// State for Vn:   Valves[(2n-2)] (timeout), Valves[(2n-1)] (discrepancy)
// Device:         USInt#n

// V15: AlarmID timeout = 110029, disc = 110030; Valves[28], Valves[29]; Device = USInt#15
```

**Body — system calls:**
```scl
_hSysEstop(IsActive := NOT DB_SYS.EStop.StOK,
            AlarmID := DInt#110031, Category := USInt#2, Severity := USInt#3,
            Device := USInt#0, CmdAck := FALSE, EnableHistory := EnableHistory,
            State := DB_ALARMS.System[0], HistBuf := DB_HIST.Buf);

_hSysAir(IsActive := NOT DB_SYS.AirOK,
          AlarmID := DInt#110032, Category := USInt#2, Severity := USInt#3,
          Device := USInt#0, CmdAck := FALSE, EnableHistory := EnableHistory,
          State := DB_ALARMS.System[1], HistBuf := DB_HIST.Buf);

_hSysHold(IsActive := DB_SEQUENCES.RunnerCtrl.Faults.FltAny,
           AlarmID := DInt#210001, Category := USInt#2, Severity := USInt#1,
           Device := USInt#0, CmdAck := FALSE, EnableHistory := EnableHistory,
           State := DB_ALARMS.System[2], HistBuf := DB_HIST.Buf);

_hSysStepTout(IsActive := DB_SEQUENCES.RunnerCtrl.Faults.FaultCode = USInt#1,
               AlarmID := DInt#210002, Category := USInt#2, Severity := USInt#1,
               Device := USInt#0, CmdAck := FALSE, EnableHistory := EnableHistory,
               State := DB_ALARMS.System[3], HistBuf := DB_HIST.Buf);

_hSysSeqStart(IsActive := DB_SEQUENCES.RunnerCtrl.Status.StExecute,
               AlarmID := DInt#310001, Category := USInt#2, Severity := USInt#0,
               Device := USInt#0, CmdAck := FALSE, EnableHistory := EnableHistory,
               State := DB_ALARMS.System[4], HistBuf := DB_HIST.Buf);

_hSysSeqStop(IsActive := DB_SEQUENCES.RunnerCtrl.Status.StStopped,
              AlarmID := DInt#310002, Category := USInt#2, Severity := USInt#0,
              Device := USInt#0, CmdAck := FALSE, EnableHistory := EnableHistory,
              State := DB_ALARMS.System[5], HistBuf := DB_HIST.Buf);

_hSysWdg(IsActive := SYS_WDG_FLT,
          AlarmID := DInt#190001, Category := USInt#2, Severity := USInt#3,
          Device := USInt#0, CmdAck := FALSE, EnableHistory := EnableHistory,
          State := DB_ALARMS.System[6], HistBuf := DB_HIST.Buf);
```

**Body — global commands (rising edge, using statics):**
```scl
// CmdAckAll — acknowledge all active alarm states
IF CmdAckAll AND NOT _prevAckAll THEN
    FOR _i := 0 TO 29 DO
        DB_ALARMS.Valves[_i].Acknowledged := TRUE;
    END_FOR;
    FOR _i := 0 TO 6 DO
        DB_ALARMS.System[_i].Acknowledged := TRUE;
    END_FOR;
END_IF;
_prevAckAll := CmdAckAll;

// CmdClear — reset history ring buffer
IF CmdClear AND NOT _prevClear THEN
    DB_HIST.Buf.Head  := 0;
    DB_HIST.Buf.Tail  := 0;
    DB_HIST.Buf.Count := 0;
    DB_HIST.Buf.Full  := FALSE;
    FOR _i := 0 TO 63 DO
        DB_HIST.Buf.Entries[_i].Valid := FALSE;
    END_FOR;
END_IF;
_prevClear := CmdClear;
```

> Note: `_hSysSeqStart` and `_hSysSeqStop` raise I-level events on rising edges of their
> IsActive signals (detected internally by FB_AlarmHandler `_PrevActive`). Since `StExecute`
> and `StStopped` are level signals, each rising edge (transition into EXECUTE or STOPPED)
> produces exactly one history entry. This is the intended behavior for sequence event logging.

---

## 9. Tag Naming — T-STD-001

Following `naming-conventions.md`. Area code for valve clusters: `VALV`.

### 9.1 I/O tags (PLC tag table: `VALV_IO_Tags`, linked to DB_VALV_IO members)

```
VALV_V[nn]_OPEN      : BOOL  %Q     Solenoid A — open command          (nn = 01..15)
VALV_V[nn]_CLOSE     : BOOL  %Q     Solenoid B — close (double acting)
VALV_V[nn]_AO_CMD    : INT   %QW    AO raw command (modulating only)
VALV_V[nn]_FBK_OP    : BOOL  %I     Position sensor — open
VALV_V[nn]_FBK_CL    : BOOL  %I     Position sensor — closed
VALV_V[nn]_AI_POS    : INT   %IW    AI raw position (modulating only)
```

These tags are mapped to `DB_VALV_IO.IO[nn]` members in the I/O copy block at the top and bottom of OB1.

### 9.2 System tags (PLC tag table: `SYS_Tags`)

```
SYS_AIR_OK           : BOOL  %I     Compressed air pressure OK (physical DI)
SYS_ESTOP_RELAY      : BOOL  %I     Safety relay monitored output
SYS_RESET_BTN        : BOOL  %I     Physical reset button
SYS_WATCHDOG_CNT     : DINT         Watchdog counter (incremented in OB30)
SYS_WDG_FLT          : BOOL         Watchdog fault — set when counter stops (see Section 10.4)
SYS_RUN              : BOOL         PLC in run state
```

DB members accessed by block name: `DB_SYS.EStop.StOK`, `DB_VALV_CTRL.CTRL[n].StOpen`, etc.

---

## 10. Block Tree and Startup

### 10.1 Block tree (TIA Portal)

Block numbers follow `naming-conventions.md` Section 8. Instance DBs are auto-assigned
by TIA Portal — do not manually assign instance DB numbers.

```
Program blocks/
├── _Main/
│   ├── OB1   — Main [LAD]
│   ├── OB100 — Startup [SCL]
│   └── OB30  — Cyclic interrupt 100ms [LAD]
│
├── SEQ/
│   ├── FB_SeqRunner    [SCL]  FB300    — T-MOD-032 (standalone runner)
│   └── FB_SeqMaster    [SCL]  FB310    — T-MOD-032 (master; not called in T-STD-001)
│
├── VALV/
│   ├── FB_VALV_OnOff   [LAD]  FB200    — T-MOD-002
│   └── FB_VALV_Ctrl    [LAD]  FB210    — T-MOD-003 (optional)
│
├── ALARM/
│   ├── FB_AlarmHandler [SCL]  FB700    — T-MOD-018a
│   └── FB_AlarmManager [SCL]  FB710    — T-MOD-018b
│
├── Common/
│   ├── FB_DI           [LAD]  FB900    — T-MOD-024
│   ├── FB_DO           [LAD]  FB910    — T-MOD-025
│   ├── FB_ESTOP        [LAD]  FB400    — T-MOD-008
│   ├── FB_MOTOR        [LAD]  FB100    — T-MOD-001
│   └── FB_TIMER_EXT    [LAD]  FB1000   — T-MOD-031
│
└── Global DBs/
    ├── DB_VALV_IO      [non-optimized]  DB9010
    ├── DB_VALV_CTRL    [optimized]      DB9011
    ├── DB_SEQUENCES    [optimized]      DB9012
    ├── DB_SYS          [optimized]      DB9013
    ├── DB_ALARMS       [optimized]      DB9014
    ├── DB_MOTOR_IO     [non-optimized]  DB9015
    ├── DB_HIST         [optimized]      DB9016
    └── DB_DEVICES      [optimized]      DB9017
```

**Instance DBs (auto-assigned by TIA Portal — naming convention):**

| FB instance | iDB name | Notes |
|---|---|---|
| FB_MOTOR | iDB_Motor | Unique |
| FB_VALV_OnOff × n | iDB_ValvOnOff_01 .. _15 | One per valve |
| FB_VALV_Ctrl × n | iDB_ValvCtrl_01 .. _n | Optional; one per modulating valve |
| FB_SeqRunner | iDB_SeqRunner | Unique |
| FB_AlarmManager | iDB_AlarmManager | Unique; contains 37 FB_AlarmHandler sub-instances |
| FB_ESTOP | iDB_EStop | Unique |
| FB_DI × n | iDB_DI_AirOK, iDB_DI_ResetBtn, … | One per DI |
| FB_DO × n | iDB_DO_Beacon, … | One per DO |

Note: FB_AlarmHandler has **no individual iDBs** — declared as multi-instance VAR inside
FB_AlarmManager. All 37 handlers share `iDB_AlarmManager`.

### 10.2 OB1 call order

```
// ── INPUT COPY ─────────────────────────────────────────────────────────────────
// Copy physical %I tags → DB_VALV_IO.IO[n] and DB_MOTOR_IO.IO input members (inline LAD)

1. FB_ESTOP            — E-stop chain; writes DB_SYS.EStop.StOK

2. FB_DI (aux inputs)  — Debounce SYS_AIR_OK, operator buttons
                         Example: FB_DI(PhysIn:=SYS_AIR_OK, Invert:=FALSE, Out:=DB_SYS.AirOK)

// Wire interlocks inline (LAD) before calling FB_SeqRunner:
// DB_SEQUENCES.RunnerCtrl.Interlocks.IntlkEstop := DB_SYS.EStop.StOK
// DB_SEQUENCES.RunnerCtrl.Interlocks.IntlkReady :=
//     DB_SYS.EStop.StOK AND DB_SYS.AirOK AND NOT DB_SYS.Motor.FltAny

3. FB_MOTOR            — Hydraulic/pneumatic pump
                         IN_OUT IO   := DB_MOTOR_IO.IO
                         IN_OUT Ctrl := DB_SYS.Motor
                         IN_OUT Dev  := DB_DEVICES.Motors[1]

4. FB_SeqRunner        — PackML sequence runner
                         IN_OUT Ctrl    := DB_SEQUENCES.RunnerCtrl
                         IN_OUT Devices := DB_DEVICES.Valves         // ARRAY[1..32]
                         IN_OUT Steps   := DB_SEQUENCES.SeqSteps
                         IN_OUT ExtCond := <OB1 TEMP ARRAY[1..8] OF BOOL — stub>
                         IN_OUT SubDone, SubStart, SubHeld, SubPaused := <OB1 TEMP stubs>
                         IN     nDevices := <project constant; 1..15>
                         IN     nSteps   := DB_SEQUENCES.nSteps

5. FB_VALV_OnOff[1..n] — Individual on/off valve FBs
                         IN_OUT IO   := DB_VALV_IO.IO[n]
                         IN_OUT Ctrl := DB_VALV_CTRL.CTRL[n]
                         IN_OUT Dev  := DB_DEVICES.Valves[n]

6. FB_VALV_Ctrl[1..n]  — Individual modulating valve FBs (if present; same pattern)
                         IN_OUT IO   := DB_VALV_IO.IO[n]
                         IN_OUT Ctrl := DB_VALV_CTRL.CTRL[n]
                         IN_OUT Dev  := DB_DEVICES.Valves[n]

7. FB_DO (aux outputs) — Write auxiliary DO tags

8. Watchdog check      — SYS_WDG_FLT := (DB_SYS.WdgSnapshot = DB_SYS.WdgPrev);
                         DB_SYS.WdgPrev := DB_SYS.WdgSnapshot;
                         DB_SYS.WdgSnapshot := SYS_WATCHDOG_CNT;

9. FB_AlarmManager     — 37 alarm instances + global commands
                         IN CmdAckAll     := DB_SYS.AlarmCmdAckAll
                         IN CmdClear      := DB_SYS.AlarmCmdClear
                         IN EnableHistory := TRUE

// ── OUTPUT COPY ────────────────────────────────────────────────────────────────
// Copy DB_VALV_IO.IO[n] and DB_MOTOR_IO.IO output members → physical %Q tags (inline LAD)
```

Rationale: Inputs first (safety chain before everything), interlocks wired before runner,
valve FBs called after runner (so Dev.Commands.CmdAuto from this scan drives valve outputs
in the same scan), alarm manager last (captures final scan state of all fault flags).

### 10.3 OB100 Startup

OB100 executes once at CPU startup. Language: SCL (justified: FOR loops over arrays).

```scl
// 1. Initialize all valve commands to safe state
FOR n := 1 TO 15 DO
    DB_VALV_CTRL.CTRL[n].CmdManOpen   := FALSE;
    DB_VALV_CTRL.CTRL[n].CmdManClose  := FALSE;
    DB_VALV_CTRL.CTRL[n].ModeManual   := FALSE;
    DB_VALV_CTRL.CTRL[n].CmdReset     := FALSE;
END_FOR;

// 2. Initialize sequence runner
DB_SEQUENCES.RunnerCtrl.Status.State     := USInt#0;    // ST_IDLE
DB_SEQUENCES.RunnerCtrl.Status.StStep    := 1;
DB_SEQUENCES.RunnerCtrl.Faults.FltAny   := FALSE;
DB_SEQUENCES.RunnerCtrl.Faults.FaultCode := USInt#0;
// Config (StopTimeout, HoldTimeout, AutoResume) set here if not in DB initial values:
DB_SEQUENCES.RunnerCtrl.Config.StopTimeout := 5.0;
DB_SEQUENCES.RunnerCtrl.Config.HoldTimeout := 30.0;

// 3. Initialize alarm history
DB_HIST.Buf.Head    := 0;
DB_HIST.Buf.Tail    := 0;
DB_HIST.Buf.Count   := 0;
DB_HIST.Buf.MaxSize := 64;    // Profile B — must match ARRAY[0..63] in UDT_AlarmHistBuffer
DB_HIST.Buf.Full    := FALSE;

// 4. Initialize DB_DEVICES (enable only installed positions; DB initial values do this too)
FOR n := 1 TO 32 DO
    DB_DEVICES.Valves[n].Config.Enabled := FALSE;    // DB initial values set TRUE for 1..nValves
END_FOR;
FOR n := 1 TO 16 DO
    DB_DEVICES.Motors[n].Config.Enabled := FALSE;    // DB initial values set TRUE for [1] if motor present
END_FOR;
```

Valve configuration parameters (`ToutOpen`, `ToutClose`, `Discrepancy`, `FbkType`, `ValveType`,
`FltSeverity`, `Enabled`, `SimMode`, `SimDelay`) are **not** initialized in OB100 — they are
pre-set in the DB initial values at engineering time. The implementation checklist must verify
these are correct before first run.

> Note: OB100 `FOR` loops for DB_DEVICES are a safety fallback. DB initial values (set in TIA
> Portal's "Start values" column) take effect on power-up before OB100 runs and are the
> authoritative source for device configuration.

### 10.4 OB30 Cyclic Interrupt (100ms)

OB30 increments `SYS_WATCHDOG_CNT` once per 100ms cycle:
```
SYS_WATCHDOG_CNT := SYS_WATCHDOG_CNT + 1
```

**Fault detection (in OB1, step 8 of call order):**

OB1 captures two consecutive snapshots. If they are equal, OB30 has stopped running:
```
SYS_WDG_FLT        := (DB_SYS.WdgSnapshot = DB_SYS.WdgPrev)
DB_SYS.WdgPrev     := DB_SYS.WdgSnapshot
DB_SYS.WdgSnapshot := SYS_WATCHDOG_CNT
```

`SYS_WDG_FLT` (BOOL) is the alarm source for F9-0001. The HMI S004 screen also displays
`SYS_WATCHDOG_CNT` as a live counter — if it freezes during normal operation, WinCC raises
a communication alarm independently of the PLC-side fault.

Note: in normal operation, `WdgSnapshot` and `WdgPrev` differ every scan because OB30 fires
every 100ms and OB1 scan time is < 10ms. If OB30 stops (CPU diagnostic, memory fault),
two consecutive OB1 scans will see identical snapshots, setting `SYS_WDG_FLT`.

---

## 11. Safety Architecture

ISO 13849-1. Typical risk graph for valve cluster: S1/F1/P1 → **PLc required**.

**Architecture: Category 2 with safety relay**

- Single E-stop button with two mechanically linked NC contacts (dual channel input)
- Safety relay: **Pilz PNOZ XV2P** or equivalent (self-monitoring, Cat. 2/3 rated relay)
- Safety relay OUTPUT: wired to PLC standard DI (monitoring → FB_ESTOP) AND to power cut relay for solenoid supply (hardware de-energization ≤ 20ms)
- No F-CPU required for PLc on this machine type
- Reset: physical illuminated reset button hardwired to safety relay reset input

**Safety function definition:**
```
SF-001: E-stop — de-energizes all valve solenoids
  PLr: PLc | Architecture: Cat. 2 | Hardware: Pilz PNOZ XV2P
  Response time: T_relay(≈15ms) + T_contactors(≈10ms) = ≤ 25ms
```

**Risk assessment placeholder:** `docs/safety/risk-assessment.xlsx` — must be completed before implementation begins (mandatory per `safety.md`).

---

## 12. Network Configuration

Network state: State 1 (isolated machine, standalone).

```
192.168.1.1    plc-01     S7-1214C DC/DC/DC
192.168.1.10   hmi-01     KTP700 Basic
192.168.1.50   (optional) Remote access — Secomea SiteManager or equivalent
```

PROFINET device names (set in TIA Portal hardware config; must match physical device names before going online): `plc-01`, `hmi-01`.

**Security baseline (SL 1 per IEC 62443 — isolated machine):**
- No default passwords on PLC, HMI, or managed switch
- OPC-UA anonymous access: DISABLED
- PLC web server and FTP: DISABLED
- Unused PROFINET ports on switch: DISABLED
- Credential details: `docs/network/opc-ua-config.md` (private repo)

---

## 13. OPC-UA (T-MOD-019)

S7-1214C firmware V4.x supports the integrated OPC-UA server. No separate FB — configured in TIA Portal hardware configuration.

**TIA Portal setup:**
- Enable OPC-UA server in CPU properties
- Security policy: `Basic256Sha256 — Sign & Encrypt` (minimum)
- Anonymous access: **DISABLED** (mandatory per `network-standards.md`)
- Port: 4840
- Namespace: `MATESTAIN.T-STD-001`

**Node structure (configured via OPC-UA server interface in TIA Portal):**
```
T-STD-001/
  Sequence/
    RunnerCtrl   ← DB_SEQUENCES.RunnerCtrl (read/write)
  Valve/
    V01/Ctrl     ← DB_VALV_CTRL.CTRL[1] (read/write)
    V01/IO       ← DB_VALV_IO.IO[1] (read-only)
    ...
    V15/Ctrl     ← DB_VALV_CTRL.CTRL[15] (read/write)
    V15/IO       ← DB_VALV_IO.IO[15] (read-only)
  System/
    EStop        ← DB_SYS.EStop (read-only)
    AirOK        ← DB_SYS.AirOK (read-only)
    Run          ← SYS_RUN (read-only)
  Alarms/
    Active/
      Valves     ← DB_ALARMS.Valves (read-only — O(1) per-alarm access)
      System     ← DB_ALARMS.System (read-only)
    History/
      Buf        ← DB_HIST.Buf (read-only — ring buffer for Plantwise polling)
    CmdAckAll    ← DB_SYS.AlarmCmdAckAll (write)
    CmdClear     ← DB_SYS.AlarmCmdClear  (write — supervisor only)
```

`AlarmCmdAckAll` and `AlarmCmdClear` are rising-edge triggered inside FB_AlarmManager
(via static `_prevAckAll`/`_prevClear`). Plantwise writes TRUE; the PLC processes it on the
next scan. The PLC does NOT reset these flags — Plantwise is responsible for resetting to
FALSE after acknowledgement.

---

## 14. HMI Design

Platform: KTP700 Basic 6" (800×480), WinCC Basic, L1 level — max 10 screens.

### 14.1 Screen list

| ID | Name | Access | Purpose |
|---|---|---|---|
| S001 | Overview | Operator | Machine state, E-stop, sequence run/stop, valve matrix summary |
| S002 | Valve Detail | Operator | Per-valve status and individual manual commands |
| S003 | Alarms | Operator | Active alarm list |
| S004 | System | Supervisor | E-stop detail, air pressure, watchdog, PackML state, OPC-UA state |

### 14.2 S001 — Overview

```
┌─────────────────────────────────────────────┐
│ T-STD-001 — Valve Cluster       12:34  [!2] │
├─────────────────────────────────────────────┤
│  ● FAULT — Valve V03 Timeout               │  ← alarm banner (RED, conditional)
├────────────────┬────────────────────────────┤
│ SEQUENCE       │  VALVE MATRIX              │
│ ○ IDLE         │  V01 ■CLOSED  V09 ■CLOSED  │
│ Step: —        │  V02 ■CLOSED  V10 ■CLOSED  │
│                │  V03 ●FAULT   V11 ■CLOSED  │
│ [▶ START]      │  V04 ■CLOSED  V12 ○OPEN    │
│ [■ STOP ]      │  V05 ■CLOSED  V13 ■CLOSED  │
│ [↺ RESET]      │  V06 ■CLOSED  V14 ■CLOSED  │
│                │  V07 ■CLOSED  V15 ■CLOSED  │
│ E-STOP: OK     │  V08 ■CLOSED              │
│ AIR:    OK     │                            │
└────────────────┴────────────────────────────┘
│ [Home] [Valves] [Alarms ^2]  [System]       │
└─────────────────────────────────────────────┘
```

**S001 tag bindings:**

| Element | PLC Tag | R/W |
|---|---|---|
| Sequence Execute / Stopped / Fault | DB_SEQUENCES.RunnerCtrl.Status.StExecute / StStopped / Faults.FltAny | R |
| Sequence step | DB_SEQUENCES.RunnerCtrl.Status.StStep | R |
| START button | DB_SEQUENCES.RunnerCtrl.Commands.CmdStart | W (rising edge) |
| STOP button | DB_SEQUENCES.RunnerCtrl.Commands.CmdStop | W (rising edge) |
| RESET button | DB_SEQUENCES.RunnerCtrl.Commands.CmdReset | W (rising edge) |
| E-stop state | DB_SYS.EStop.StOK | R |
| Air OK | DB_SYS.AirOK | R |
| V[nn] state chips | DB_VALV_CTRL.CTRL[nn].State | R (maps to color per Section 14.6) |

### 14.3 S002 — Valve Detail

One screen with valve index selector (1–15). Tag indirect addressing via indexed tag arrays (WinCC Basic).

```
┌─────────────────────────────────────────────┐
│ <- Valve Detail          V[03]  < > [1..15] │
├─────────────────────────────────────────────┤
│ TAG: VALV_V03  TYPE: OnOff_Double           │
│                                             │
│ +-- STATUS ─────────────────────────────+  │
│ | State: ● FAULT — Timeout              |  │
│ | Command:   OPEN (auto)                |  │
│ | FbkOpen=0  FbkClosed=0                |  │
│ | Fault: TIMEOUT — 2.0s exceeded        |  │
│ +────────────────────────────────────────+  │
│                                             │
│ [MANUAL MODE]  <- toggle (Supervisor only)  │
│                                             │
│ [OPEN]  [CLOSE]   (active only in Manual)   │
│ [RESET FAULT]                               │
└─────────────────────────────────────────────┘
│ [Home] [Valves] [Alarms ^2]  [System]       │
└─────────────────────────────────────────────┘
```

**S002 tag bindings (valve index n):**

| Element | PLC Tag | R/W |
|---|---|---|
| Valve type | DB_VALV_CTRL.CTRL[n].ValveType | R |
| State display | DB_VALV_CTRL.CTRL[n].State | R |
| StOpen / StClosed / StMoving | DB_VALV_CTRL.CTRL[n].StOpen / StClosed / StMoving | R |
| FltTimeout / FltDiscrepancy | DB_VALV_CTRL.CTRL[n].FltTimeout / FltDiscrepancy | R |
| MANUAL MODE toggle | DB_VALV_CTRL.CTRL[n].ModeManual | W (Supervisor only) |
| OPEN button | DB_VALV_CTRL.CTRL[n].CmdManOpen | W (manual mode only) |
| CLOSE button | DB_VALV_CTRL.CTRL[n].CmdManClose | W (manual mode only) |
| RESET button | DB_VALV_CTRL.CTRL[n].CmdReset | W |

OPEN and CLOSE buttons are enabled in WinCC only when `DB_VALV_CTRL.CTRL[n].ModeManual = TRUE`. Supervisor access level required to toggle manual mode.

### 14.4 S003 — Alarms

> **Two independent alarm channels — do not confuse them:**
> WinCC Basic has its own built-in alarm system configured in the HMI alarm editor. Alarm
> triggers are bound directly to PLC fault tags (`DB_VALV_CTRL.CTRL[n].FltTimeout`, etc.).
> `DB_ALARMS` and `DB_HIST` are NOT read by WinCC — they are the structured alarm store
> for Plantwise/OPC-UA polling only.
> WinCC = on-site operator display. DB_ALARMS/DB_HIST = remote system integration.

WinCC Basic alarm system configuration:
- Alarm class FAULT (RED): ACK required. Triggers: all `FltTimeout`, `FltDiscrepancy`, `FltChain`, `FltAny` tags.
- Alarm class WARNING (AMBER): ACK optional. Trigger: `DB_SEQUENCES.RunnerCtrl.Faults.FltAny`.
- Alarm class INFO (GRAY-BLUE): no ACK. Triggers: `DB_SEQUENCES.RunnerCtrl.Status.StExecute` rising, `StStopped` rising.

Alarm text and IDs match exactly the alarm list in Section 15.

```
┌─────────────────────────────────────────────┐
│ <- Alarms (3 active)           [ACK ALL]    │
├──────────┬──────────────────────────────────┤
│ ACTIVE(3)│ F  F1-0005  V03 Timeout fault    │ <- RED
│          │             12:31:45    [ACK]    │
│          │ W  W1-0001  Sequence in hold     │ <- AMBER
│          │             12:31:45    [ACK]    │
│          │ F  F1-0031  E-stop activated     │ <- RED
│          │             12:28:03    [ACK]    │
│          │                                  │
│ HISTORY  │ (last 20 per WinCC Basic limit)  │
└──────────┴──────────────────────────────────┘
│ [Home] [Valves] [Alarms ^3]  [System]       │
└─────────────────────────────────────────────┘
```

### 14.5 S004 — System (Supervisor access)

```
┌─────────────────────────────────────────────┐
│ <- System                [Supervisor only]  │
├─────────────────────────────────────────────┤
│ E-STOP                                      │
│   Status: OK / TRIPPED / RESET REQUIRED     │
│   Chain fault: NO                           │
│                              [RESET]        │
│                                             │
│ UTILITIES                                   │
│   Compressed air: OK / LOW                  │
│   Motor (pump):   RUNNING / STOPPED / DISABLED │
│                                             │
│ SEQUENCE                                    │
│   PackML State: IDLE / EXECUTE / HELD / …  │
│   Step: 3   Fault code: 0                   │
│                                             │
│ PLC SYSTEM                                  │
│   CPU Run: YES    Watchdog: 000123          │
│   OPC-UA:  CONNECTED / DISCONNECTED         │
│   Firmware: V4.x  IP: 192.168.1.1          │
└─────────────────────────────────────────────┘
│ [Home] [Valves] [Alarms ^3]  [System]       │
└─────────────────────────────────────────────┘
```

**S004 tag bindings:**

| Element | PLC Tag | R/W |
|---|---|---|
| E-stop StOK / StTripped / StResetReqd | DB_SYS.EStop.StOK / StTripped / StResetReqd | R |
| E-stop RESET button | DB_SYS.EStop reset mapped via FB_ESTOP CmdReset | W (Supervisor) |
| Air OK | DB_SYS.AirOK | R |
| Motor State | DB_SYS.Motor.State | R |
| Sequence PackML State | DB_SEQUENCES.RunnerCtrl.Status.State | R (text mapped: 0=IDLE, 1=STARTING, 2=EXECUTE, …) |
| Sequence Step | DB_SEQUENCES.RunnerCtrl.Status.StStep | R |
| Sequence FaultCode | DB_SEQUENCES.RunnerCtrl.Faults.FaultCode | R |
| Watchdog counter | SYS_WATCHDOG_CNT | R |
| OPC-UA state | CPU diagnostics tag (WinCC system tag) | R |

PackML State display: WinCC text list mapping State (USInt) to name strings:
0=IDLE, 1=STARTING, 2=EXECUTE, 3=COMPLETING, 4=COMPLETE, 5=HOLDING, 6=HELD, 7=RESUMING,
8=PAUSING, 9=PAUSED, 11=STOPPING, 12=STOPPED, 13=ABORTING, 14=ABORTED.

### 14.6 Valve faceplate specification

Follows `hmi-scada.md` Section 5. Border-left: 4px, state-colored.
HMI evaluates `State : INT` first (single tag comparison for color mapping), then sub-conditions
for the Active state (StMoving distinguishes Moving from Open within State=2).

| State value | Condition | Display text | Color |
|---|---|---|---|
| 4 | State = 4 | DISABLED | GRAY #9E9E9E solid |
| 3 | State = 3, unACKed | FAULT | RED flash #F44336 @ 1Hz |
| 3 | State = 3, ACKed | FAULT | RED solid #B71C1C |
| 2 | State = 2, StMoving = TRUE | MOVING | CYAN flash #26C6DA @ 0.5Hz |
| 2 | State = 2, StMoving = FALSE | OPEN | GREEN #66BB6A |
| 1 | State = 1 | STANDBY | CYAN #26C6DA solid |
| 0 | State = 0 | CLOSED | GRAY-400 #BDBDBD |
| — | ModeManual = TRUE | MANUAL | YELLOW #FFEE58 (overlay on any state) |
| — | No comms | ? | PURPLE flash #AB47BC |

Icon: valve body symbol (T-shape for 2-way on/off, cross or Y-shape for 3-position modulating). Use Material Symbols "valve" or closest equivalent available in WinCC Basic.

---

## 15. Alarm List

Following `alarm-management.md`. Domain 1 (PLC). Profile B (dynamic circular list, 64 entries).
15 valves × 2 faults + 7 cluster/system alarms = **37 total alarms**.

### Valve faults (per valve, nn = 01..15)

| Alarm ID | AlarmID DInt | Severity | Sub-cat | Text | PLC Tag (source) |
|---|---|---|---|---|---|
| F1-0001 | 110001 | F | FC | Valve V01 — Timeout fault. Did not reach commanded position. | DB_VALV_CTRL.CTRL[1].FltTimeout |
| F1-0002 | 110002 | F | FC | Valve V01 — Discrepancy fault. Feedback does not match command. | DB_VALV_CTRL.CTRL[1].FltDiscrepancy |
| F1-0003 | 110003 | F | FC | Valve V02 — Timeout fault. | DB_VALV_CTRL.CTRL[2].FltTimeout |
| F1-0004 | 110004 | F | FC | Valve V02 — Discrepancy fault. | DB_VALV_CTRL.CTRL[2].FltDiscrepancy |
| … | … | | | Pattern: F1-[2n-1]=timeout Vn, F1-[2n]=discrepancy Vn | |
| F1-0029 | 110029 | F | FC | Valve V15 — Timeout fault. | DB_VALV_CTRL.CTRL[15].FltTimeout |
| F1-0030 | 110030 | F | FC | Valve V15 — Discrepancy fault. | DB_VALV_CTRL.CTRL[15].FltDiscrepancy |

Sub-category FC = Fault Critical (controlled stop required). DB_ALARMS slot: Valve n → Valves[(2n-2)] for timeout, Valves[(2n-1)] for discrepancy.

### Cluster and system faults

| Alarm ID | AlarmID DInt | Severity | Sub-cat | Text | PLC Tag (source) |
|---|---|---|---|---|---|
| F1-0031 | 110031 | F | FS | Cluster — E-stop activated. Reset and restart required. | NOT DB_SYS.EStop.StOK |
| F1-0032 | 110032 | F | FC | Cluster — Compressed air pressure low. Check supply. | NOT DB_SYS.AirOK |
| W1-0001 | 210001 | W | — | Cluster — Sequence in hold. Device fault detected. | DB_SEQUENCES.RunnerCtrl.Faults.FltAny |
| W1-0002 | 210002 | W | — | Cluster — Sequence step timeout. Check device and advance condition. | DB_SEQUENCES.RunnerCtrl.Faults.FaultCode = USInt#1 |
| I1-0001 | 310001 | I | — | Cluster — Sequence started. | DB_SEQUENCES.RunnerCtrl.Status.StExecute (rising edge) |
| I1-0002 | 310002 | I | — | Cluster — Sequence stopped. | DB_SEQUENCES.RunnerCtrl.Status.StStopped (rising edge) |
| F9-0001 | 190001 | F | FS | System — PLC watchdog timeout. Cycle power after inspection. | SYS_WDG_FLT |

Sub-category FS = Fault Safety (immediate stop, requires supervisor reset).
DB_ALARMS slots: System[0]=F1-0031, [1]=F1-0032, [2]=W1-0001, [3]=W1-0002, [4]=I1-0001, [5]=I1-0002, [6]=F9-0001.

**Total alarms: 37.** Well within Profile B capacity (64 entries).

> W1-0001 source: `DB_SEQUENCES.RunnerCtrl.Faults.FltAny` is TRUE in HOLDING, ABORTING, ABORTED.
> W1-0002 source: `FaultCode = USInt#1` means a step timeout caused the abort — evaluated by
> FB_AlarmHandler's IsActive comparison each scan. This is a boolean expression passed to IsActive.

---

## 16. Open Design Decisions

Items that require a per-project decision before or during TIA Portal implementation.

| # | Item | Default / Status | Required action |
|---|---|---|---|
| 1 | Safe state of double acting valves on fault | De-energize both solenoids (hold last position) | Verify per P&ID — some installations require forced close |
| 2 | FC_ALARM_COLLECT calling pattern | **Resolved** — FB_AlarmManager (FB710) in 1 OB1 call | No action required |
| 3 | Analog scaling for FbkAnalog | **Resolved** | Verify sensor signal range against electrical drawings before commissioning |
| 4 | Sequence step array size | 30 steps max | Increase if project requires more than 30 commanded steps |
| 5 | Motor FB inclusion | **Resolved** — always include; set Enabled=FALSE if no motor | No action required |
| 6 | SimMode activation scope | Per valve (current design: SimMode in UDT_ValveCtrl per valve) | Consider a global DB_SYS.SimModeAll BOOL that overrides all valve SimMode flags simultaneously — faster for full-FAT. Add to DB_SYS and check in OB1 before each valve call if adopted. |
| 7 | Enabled default for unused valve positions | TRUE for 1..nActualValves, FALSE for the rest | Set DB initial values explicitly for DB_VALV_CTRL.CTRL[n].Enabled AND DB_DEVICES.Valves[n].Config.Enabled. Verify in checklist that no disabled valve appears in any SeqStep ActMask. |
| 8 | W1-0002 alarm source | **Resolved** — DB_SEQUENCES.RunnerCtrl.Faults.FaultCode = USInt#1 | No action required |
| 9 | SubDone/SubStart/SubHeld/SubPaused in FB_SeqRunner | In T-STD-001 (single runner, no sub-sequences) these are stubs | Declare as TEMP arrays in OB1 for T-STD-001. For future projects with FB_SeqMaster, wire to actual sub-sequence Ctrl arrays. |
| 10 | ActSetpoint[1..15] memory in UDT_SeqStep | 15 REALs × 4 bytes × 30 steps = 1.8 KB — acceptable for S7-1214C (75 KB work memory) | No action for T-STD-001. For projects with no modulating valves, these elements exist but are never written — acceptable waste. |

---

## 17. Implementation Checklist

Before opening TIA Portal:

- [ ] P&ID reviewed — valve types, actuations, and safe states confirmed
- [ ] I/O list drafted — all physical addresses assigned
- [ ] Sequence steps defined and numbered (complete DB_SEQUENCES.SeqSteps initial values)
- [ ] DB initial values set for each valve: ValveType, FbkType, ToutOpen, ToutClose, Discrepancy, SimDelay, FltSeverity
- [ ] DB initial values: `DB_VALV_CTRL.CTRL[n].Enabled := TRUE` for installed valves, `FALSE` for unused positions
- [ ] DB initial values: `DB_DEVICES.Valves[n].Config.Enabled := TRUE` for 1..nValves, `FALSE` for 16..32
- [ ] DB initial values: `DB_DEVICES.Motors[1].Config.Enabled := TRUE` if pump present; `FALSE` if no motor
- [ ] DB initial values: `DB_DEVICES.Valves[n].Config.FltSeverity` — review per P&ID (default 1=Hold; set 2=Abort for critical valves)
- [ ] DB initial values: `DB_DEVICES.Motors[1].Config.FltSeverity := 2` (Abort default for motor overload)
- [ ] DB initial values: `SimMode := FALSE` for ALL valves — must be confirmed before go-live
- [ ] DB_SEQUENCES.RunnerCtrl.Config: set StopTimeout and HoldTimeout per project (defaults 5.0 / 30.0)
- [ ] DB_HIST.Buf.MaxSize := 64 set in OB100 (must match ARRAY[0..63] in UDT_AlarmHistBuffer)
- [ ] Verify SubDone/SubStart/SubHeld/SubPaused: declared as OB1 TEMP stubs or connected to sub-sequences (decision #9)
- [ ] Verify DB_ALARMS and DB_HIST: Optimized block access = ON in TIA Portal DB properties
- [ ] Open design decisions (Section 16) resolved for this specific project
- [ ] `docs/safety/risk-assessment.xlsx` completed and signed off
- [ ] Network IP assignments confirmed against `network-standards.md`
- [ ] OPC-UA credentials documented in `docs/network/opc-ua-config.md` (private repo)
- [ ] SeqStep ActMask/ActTarget/CondMask/CondTarget documented in DB comments (Bit0=V01, Bit1=V02, …)
- [ ] Verify no disabled valve (Enabled=FALSE) has its bit set in any SeqStep ActMask
- [ ] Valve sensor range confirmed against electrical drawings (4-20mA default scaling)
- [ ] nDevices constant in OB1 (passed to FB_SeqRunner) matches actual number of installed valves

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 0.3.0 | 2026-05 | Incorporated AlarmManager_TIA_V20 and SeqManager_PackML_TIA_V20 reference blocks. All reference blocks incorporated without modification: FB_AlarmHandler (FB700), FB_AlarmManager (FB710, FC→FB conversion fixes %M0.0/%M0.1 bug), FB_SeqRunner (FB300), FB_SeqMaster (FB310, documented, not called in T-STD-001). DB_DEVICES (DB9017) introduced as adapter resolving UDT_ValveCtrl/UDT_DeviceCtrl type incompatibility in TIA Portal. FB_VALV_OnOff/Ctrl/FB_MOTOR: added Dev:UDT_DeviceCtrl IN_OUT; config sync before Step 0; Step 9 adds StReady calc + Dev.Status sync. UDT_ValveCtrl: removed CmdAuto (→Dev.Commands.CmdAuto), added FltSeverity:USInt and StReady:Bool. UDT_MotorCtrl: removed CmdRun (→Dev.Commands.CmdAuto), renamed CmdManual→ModeManual, added CmdManRun, FltSeverity:USInt, StReady:Bool; fixed ToutRun_s→ToutRun inconsistency. UDT_ClusterCtrl eliminated → UDT_SeqCtrl (full PackML 14-state structure). UDT_SeqStep: superset with StepType, Timeout (was StepTout), ActSetpoint[1..15], WaitTime, ExtCondIdx, SubSeqIdx, Parallel, StepDesc. New UDTs: UDT_DeviceCtrl, UDT_SeqCtrl, UDT_MasterSubConfig, UDT_MasterCtrl, UDT_AlarmHistEntry. UDT_AlarmEntry→UDT_AlarmState (AlarmID DWORD→DInt, Acked→Acknowledged, Timestamp→TimeRaised, added TimeCleared/Count/Severity/Category/Device). UDT_AlarmBuffer→UDT_AlarmHistBuffer (added Full:Bool; Entries→UDT_AlarmHistEntry[0..63]). DB_VALV_CLUSTER→DB_SEQUENCES (DB9012): RunnerCtrl+SeqSteps+nSteps. DB_ALARM→DB_ALARMS (DB9014): Valves[0..29]+System[0..6]. New DB_HIST (DB9016), DB_DEVICES (DB9017). AlarmID encoding fixed: F=1,W=2,I=3 (eliminates v0.2.0 F/W collision); type DInt. StDone semantics documented: active state = Open/Running (not "reached commanded state"). Section 7 replaced with FB_SeqRunner+FB_SeqMaster design. Section 8.6 replaced with FB_AlarmHandler+FB_AlarmManager. OB1: interlock inline wire-up before runner; step 3 FB_MOTOR adds Dev; step 4 FB_SeqRunner replaces FB_VALV_Cluster; steps 5/6 add Dev; step 9 replaced by one FB_AlarmManager call. OB100: updated for DB_SEQUENCES/DB_HIST/DB_DEVICES init. OPC-UA: DB_VALV_CLUSTER→DB_SEQUENCES, DB_ALARM→DB_ALARMS+DB_HIST. HMI: S001 tag bindings updated; S003 alarm class sources updated; S004 adds PackML state display. Alarm list: 37 total (corrects v0.2.0 count of 36); AlarmID DInt column added; W1-0001/W1-0002/I1-0001/I1-0002 source tags updated to DB_SEQUENCES. Open decisions: #2 and #8 resolved; #9 and #10 added. Checklist: 7 new items. |
| 0.2.0 | 2026-04 | Added Section 2 — Standard Element State Encoding. UDT_ValveCtrl: added State, Enabled, SimMode, SimDelay. UDT_SeqStep: mask-based (ActMask, ActTarget, CondMask, CondTarget). UDT_MotorCtrl: formal UDT section. UDT_AlarmEntry/UDT_AlarmBuffer. FB_VALV_OnOff: Disabled check, SimMode, State calculation. FB_VALV_Ctrl: same pattern. FB_VALV_Cluster: SCL, mask-based, tStepTout → STATE_FAULT. FB_MOTOR: completed full logic, SimMode. FB_ALARM_BUFFER: updated to Buf:UDT_AlarmBuffer. OB1/OB100 updates. Alarm list: W1-0002 added → 36 total. |
| 0.1.0 | 2026-04 | Initial design — corrected REF_TO to IN_OUT, added Global DB section, all common module designs, HMI/Safety/Network/OPC-UA sections, fixed naming inconsistencies. Pass 2: block numbers, UDT_MotorIO expanded, FB_VALV_Cluster parameters moved to interface, global vs instance DB rule, watchdog fault detection, OPC-UA alarm commands, WinCC/DB_ALARM channels clarified, platform scope note. |
