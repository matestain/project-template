# Sequence Manager PackML — TIA Portal V20
**Architecture: FB_SeqRunner + FB_SeqMaster — Based on OMAC PackML / ISA-88**
Version 2.0

---

## Table of contents
1. [General architecture](#1-general-architecture)
2. [PackML — State and transition reference](#2-packml--state-and-transition-reference)
3. [UDTs](#3-udts)
4. [Global DBs](#4-global-dbs)
5. [FB_SeqRunner](#5-fb_seqrunner)
6. [FB_SeqMaster](#6-fb_seqmaster)
7. [How to set it up in TIA Portal](#7-how-to-set-it-up-in-tia-portal)
8. [How to use it — examples](#8-how-to-use-it--examples)
9. [Scaling and topologies](#9-scaling-and-topologies)
10. [Notes and limitations](#10-notes-and-limitations)

---

## 1. General architecture

```
OB1
 ├── FB_SeqMaster "Filling"
 │    ├── FB_SeqRunner [Routing_A]  ──► DB_DEVICES.Valves[1..15]
 │    ├── FB_SeqRunner [Routing_B]  ──► DB_DEVICES.Valves[1..15]
 │    └── FB_SeqRunner [Routing_C]  ──► DB_DEVICES.Valves[1..15]
 │
 ├── FB_SeqMaster "CIP"
 │    ├── FB_SeqRunner [Rinse]      ──► DB_DEVICES.Valves[1..8]
 │    └── FB_SeqRunner [Chemical]   ──► DB_DEVICES.Valves[1..8]
 │
 └── FB_SeqRunner "Agitator"        ──► DB_DEVICES.Motors[1..4]
     (standalone — no Master)
```

### Design principles

- **Complete PackML** — states, transitions, and commands per OMAC PackML v3.0 / ISA-88.
- **Structured UDT_SeqCtrl** — `Config`, `Commands`, `Status`, `Faults` sub-structs
  aligned with the project's `UDT_xxxCtrl` convention.
- **Per-device FltSeverity** — the Runner decides Hold vs Abort based on configuration,
  with no changes to existing device FBs.
- **Optional auto-resume for HELD** — configurable per Runner via `Config.AutoResume`.
- **Master response policy** — configurable per sub-sequence via `Config.SubPolicy`.

---

## 2. PackML — State and transition reference

### States

| State | Value | Description |
|--------|-------|-------------|
| IDLE | 0 | Ready, waiting for CmdStart |
| STARTING | 1 | Checking preconditions |
| EXECUTE | 2 | Running steps normally |
| COMPLETING | 3 | Last step OK, running controlled shutdown |
| COMPLETE | 4 | Sequence finished cleanly |
| HOLDING | 5 | Recoverable fault detected, stopping in a controlled manner |
| HELD | 6 | Stopped at a safe point, position preserved |
| RESUMING | 7 | Returning to EXECUTE from HELD |
| PAUSING | 8 | CmdPause received, stopping in a controlled manner |
| PAUSED | 9 | Stopped at a safe point due to a manual pause |
| RESUMING_PAUSE | 10 | Returning to EXECUTE from PAUSED (same state, different flag) |
| STOPPING | 11 | CmdStop received, controlled shutdown |
| STOPPED | 12 | Stopped cleanly, can be restarted |
| ABORTING | 13 | Critical fault, emergency shutdown |
| ABORTED | 14 | Requires CmdReset + manual verification |

### Valid transitions

```
IDLE        →  STARTING    : CmdStart + IntlkReady + IntlkEstop
STARTING    →  EXECUTE     : All preconditions OK
STARTING    →  ABORTING    : IntlkEstop = FALSE during STARTING

EXECUTE     →  HOLDING     : FltAny on a device with FltSeverity = 1 (Hold)
EXECUTE     →  ABORTING    : FltAny on a device with FltSeverity = 2 (Abort)
                             OR IntlkEstop = FALSE
EXECUTE     →  PAUSING     : CmdPause
EXECUTE     →  STOPPING    : CmdStop
EXECUTE     →  COMPLETING  : Last step completed

HOLDING     →  HELD        : All devices in a safe state
HELD        →  RESUMING    : CmdResume (always manual)
                             OR fault resolved + Config.AutoResume = TRUE
RESUMING    →  EXECUTE     : Resume conditions OK

PAUSING     →  PAUSED      : All devices in a safe state
PAUSED      →  RESUMING    : CmdResume
RESUMING    →  EXECUTE     : Conditions OK

COMPLETING  →  COMPLETE    : Controlled shutdown finished
COMPLETE    →  IDLE        : CmdReset

STOPPING    →  STOPPED     : All devices confirmed at base position
STOPPED     →  IDLE        : CmdReset

ABORTING    →  ABORTED     : Emergency shutdown completed
ABORTED     →  IDLE        : CmdReset + IntlkEstop = TRUE

ANY         →  ABORTING    : IntlkEstop = FALSE (except from ABORTED)
```

### PackML commands

| Command | Action |
|---------|--------|
| CmdStart | IDLE → STARTING |
| CmdStop | EXECUTE/PAUSED/HELD → STOPPING |
| CmdPause | EXECUTE → PAUSING |
| CmdResume | HELD/PAUSED → RESUMING |
| CmdAbort | Any state → ABORTING |
| CmdReset | COMPLETE/STOPPED/ABORTED → IDLE |
| CmdClear | Clear faults without changing state |

---

## 3. UDTs

### UDT_DeviceCtrl
> Extended with `FltSeverity` for Hold/Abort policy.

```scl
TYPE "UDT_DeviceCtrl"
    STRUCT
        // ── Config ───────────────────────────────────────────────────────
        Config : STRUCT
            DevType     : USInt;    // 0=Valve 1=Motor 2=Analog 3=Servo
            Enabled     : Bool;     // ENA — FALSE = ignored by the runner
            FltSeverity : USInt;    // 0=Warning 1=Hold 2=Abort
        END_STRUCT;

        // ── Commands (SeqRunner writes, device FB reads) ─────────────
        Commands : STRUCT
            CmdAuto     : Bool;     // Valve: Open/Close — Motor: Start/Stop
            CmdSetpoint : Real;     // Analog/Servo: target setpoint
        END_STRUCT;

        // ── Status (device FB writes, SeqRunner reads) ───────────────
        Status : STRUCT
            StReady     : Bool;     // Ready to operate
            StDone      : Bool;     // Reached the target state
            FltAny      : Bool;     // Active fault
        END_STRUCT;
    END_STRUCT;
END_TYPE
```

---

### UDT_SeqStep
> Unchanged from v1.0 — polymorphic by StepType.

```scl
TYPE "UDT_SeqStep"
    STRUCT
        StepType    : USInt;                    // 0=Action 1=Wait 2=Condition 3=SubSeq
        Timeout     : Real;                     // seconds. 0 = no timeout
        ActMask     : DWORD;
        ActTarget   : DWORD;
        ActSetpoint : ARRAY[1..32] OF Real;
        CondMask    : DWORD;
        CondTarget  : DWORD;
        WaitTime    : Real;
        ExtCondIdx  : USInt;                    // 1..8
        SubSeqIdx   : USInt;                    // 1..8
        Parallel    : Bool;
    END_STRUCT;
END_TYPE
```

---

### UDT_SeqCtrl
> The Runner's main structure. Sub-structs aligned with project convention.

```scl
TYPE "UDT_SeqCtrl"
    STRUCT

        // ── Config (written once at startup or from the HMI) ────────────
        Config : STRUCT
            AutoResume      : Bool;     // TRUE = auto-resume from HELD once the fault clears
            AutoResumeTime  : Real;     // max seconds for auto-resume (0 = no limit)
            StopTimeout     : Real;     // seconds to confirm shutdown in STOPPING/ABORTING
            HoldTimeout     : Real;     // max seconds in HOLDING before going to ABORTING
        END_STRUCT;

        // ── Commands (written by the caller or HMI) ────────────────────────
        Commands : STRUCT
            CmdStart    : Bool;
            CmdStop     : Bool;
            CmdPause    : Bool;
            CmdResume   : Bool;
            CmdAbort    : Bool;
            CmdReset    : Bool;
            CmdClear    : Bool;
        END_STRUCT;

        // ── Status (written by the Runner) ────────────────────────────────
        Status : STRUCT
            State       : USInt;    // Current PackML state (see constants)
            StStep      : Int;      // Current step
            StIdle      : Bool;
            StStarting  : Bool;
            StExecute   : Bool;
            StCompleting: Bool;
            StComplete  : Bool;
            StHolding   : Bool;
            StHeld      : Bool;
            StResuming  : Bool;
            StPausing   : Bool;
            StPaused    : Bool;
            StStopping  : Bool;
            StStopped   : Bool;
            StAborting  : Bool;
            StAborted   : Bool;
            StDone      : Bool;     // Sequence completed all steps cleanly
        END_STRUCT;

        // ── Faults (written by the Runner) ────────────────────────────────
        Faults : STRUCT
            FaultCode   : USInt;    // 0=None 1=StepTimeout 2=EStop 3=DevFault 4=HoldTimeout
            FaultStep   : Int;
            FaultDevice : Int;      // Index of the device that caused the fault
            FltAny      : Bool;
        END_STRUCT;

        // ── Interlocks (written by the caller every cycle) ─────────────────────
        Interlocks : STRUCT
            IntlkReady  : Bool;     // Process permissives OK
            IntlkEstop  : Bool;     // E-Stop OK (TRUE = no estop)
        END_STRUCT;

    END_STRUCT;
END_TYPE
```

---

### UDT_MasterSubConfig
> Master's response policy to sub-sequence faults.

```scl
TYPE "UDT_MasterSubConfig"
    STRUCT
        OnFaultPolicy   : USInt;    // 0=AbortAll 1=AbortOnly 2=HoldAll 3=HoldOnly
        OnTimeoutPolicy : USInt;    // same logic
        IsMandatory     : Bool;     // FALSE = this runner's fault doesn't affect the others
    END_STRUCT;
END_TYPE
```

---

### UDT_MasterCtrl
> Master control. Same Config/Commands/Status/Faults convention.

```scl
TYPE "UDT_MasterCtrl"
    STRUCT
        Config : STRUCT
            SubPolicy   : ARRAY[1..8] OF "UDT_MasterSubConfig";
        END_STRUCT;

        Commands : STRUCT
            CmdStart    : Bool;
            CmdStop     : Bool;
            CmdPause    : Bool;
            CmdResume   : Bool;
            CmdAbort    : Bool;
            CmdReset    : Bool;
        END_STRUCT;

        Status : STRUCT
            State       : USInt;
            StIdle      : Bool;
            StExecute   : Bool;
            StHeld      : Bool;
            StPaused    : Bool;
            StStopped   : Bool;
            StAborted   : Bool;
            StComplete  : Bool;
            StDone      : Bool;
        END_STRUCT;

        Faults : STRUCT
            FaultCode       : USInt;
            FaultSubSeq     : USInt;    // Index of the sub-sequence that caused the fault
            FltAny          : Bool;
        END_STRUCT;

        Interlocks : STRUCT
            IntlkReady  : Bool;
            IntlkEstop  : Bool;
        END_STRUCT;
    END_STRUCT;
END_TYPE
```

---

## 4. Global DBs

```scl
DATA_BLOCK "DB_DEVICES"
    VAR
        Valves  : ARRAY[1..32] OF "UDT_DeviceCtrl";
        Motors  : ARRAY[1..16] OF "UDT_DeviceCtrl";
        Analogs : ARRAY[1..8]  OF "UDT_DeviceCtrl";
        Servos  : ARRAY[1..8]  OF "UDT_DeviceCtrl";
    END_VAR
END_DATA_BLOCK

DATA_BLOCK "DB_SEQUENCES"
    VAR
        MainCtrl    : "UDT_MasterCtrl";

        Sub1Ctrl    : "UDT_SeqCtrl";
        Sub1Steps   : ARRAY[1..30] OF "UDT_SeqStep";
        nSub1Steps  : Int;

        Sub2Ctrl    : "UDT_SeqCtrl";
        Sub2Steps   : ARRAY[1..30] OF "UDT_SeqStep";
        nSub2Steps  : Int;

        Sub3Ctrl    : "UDT_SeqCtrl";
        Sub3Steps   : ARRAY[1..30] OF "UDT_SeqStep";
        nSub3Steps  : Int;
        // Add more as needed per project
    END_VAR
END_DATA_BLOCK
```

---

## 5. FB_SeqRunner

**Suggested number:** FB211

```scl
FUNCTION_BLOCK "FB_SeqRunner"

// ─────────────────────────────────────────────────────────────────────────────
// INTERFACE
// ─────────────────────────────────────────────────────────────────────────────
VAR_INPUT
    nDevices    : Int;
    nSteps      : Int;
END_VAR

VAR_IN_OUT
    Ctrl        : "UDT_SeqCtrl";
    Devices     : ARRAY[1..32] OF "UDT_DeviceCtrl";
    Steps       : ARRAY[1..30] OF "UDT_SeqStep";
    ExtCond     : ARRAY[1..8]  OF Bool;
    SubDone     : ARRAY[1..8]  OF Bool;
    SubStart    : ARRAY[1..8]  OF Bool;
    SubHeld     : ARRAY[1..8]  OF Bool;     // Feedback: sub-sequence in HELD
    SubPaused   : ARRAY[1..8]  OF Bool;     // Feedback: sub-sequence in PAUSED
END_VAR

VAR
    _prevStart      : Bool;
    _prevStop       : Bool;
    _prevPause      : Bool;
    _prevResume     : Bool;
    _prevAbort      : Bool;
    _prevReset      : Bool;
    _tStopTout      : TON;
    _tStepTout      : TON;
    _tWait          : TON;
    _tHoldTout      : TON;
    _tAutoResume    : TON;
    _stepDone       : Bool;
    _allSafe        : Bool;
    _faultDevice    : Int;
    _faultSeverity  : USInt;
    _holdFromPause  : Bool;     // Distinguish RESUMING from HELD vs PAUSED
END_VAR

VAR_TEMP
    _n          : Int;
    _bitVal     : DWORD;
END_VAR

VAR CONSTANT
    // PackML states
    ST_IDLE         : USInt := 0;
    ST_STARTING     : USInt := 1;
    ST_EXECUTE      : USInt := 2;
    ST_COMPLETING   : USInt := 3;
    ST_COMPLETE     : USInt := 4;
    ST_HOLDING      : USInt := 5;
    ST_HELD         : USInt := 6;
    ST_RESUMING     : USInt := 7;
    ST_PAUSING      : USInt := 8;
    ST_PAUSED       : USInt := 9;
    ST_STOPPING     : USInt := 11;
    ST_STOPPED      : USInt := 12;
    ST_ABORTING     : USInt := 13;
    ST_ABORTED      : USInt := 14;
END_VAR

// ─────────────────────────────────────────────────────────────────────────────
// BODY
// ─────────────────────────────────────────────────────────────────────────────

// ── CmdAbort and E-Stop: highest priority, any state ────────────────────
IF (Ctrl.Commands.CmdAbort AND NOT _prevAbort)
   OR (NOT Ctrl.Interlocks.IntlkEstop AND Ctrl.Status.State <> ST_ABORTED) THEN
    Ctrl.Status.State       := ST_ABORTING;
    Ctrl.Faults.FaultCode   := USInt#2;   // EStop/Abort
    Ctrl.Faults.FltAny      := TRUE;
END_IF;

// ── PackML state machine ────────────────────────────────────────────────
CASE Ctrl.Status.State OF

    // ── IDLE ─────────────────────────────────────────────────────────────────
    ST_IDLE:
        Ctrl.Status.StStep  := 1;
        Ctrl.Status.StDone  := FALSE;
        Ctrl.Faults.FltAny  := FALSE;
        Ctrl.Faults.FaultCode := USInt#0;

        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                Devices[_n].Commands.CmdAuto     := FALSE;
                Devices[_n].Commands.CmdSetpoint := 0.0;
            END_IF;
        END_FOR;

        IF Ctrl.Commands.CmdStart AND NOT _prevStart
           AND Ctrl.Interlocks.IntlkReady
           AND Ctrl.Interlocks.IntlkEstop THEN
            Ctrl.Status.State := ST_STARTING;
        END_IF;

    // ── STARTING ─────────────────────────────────────────────────────────────
    ST_STARTING:
        // Check that all enabled devices have no faults
        // and are ready to operate
        _allSafe := TRUE;
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                IF Devices[_n].Status.FltAny OR NOT Devices[_n].Status.StReady THEN
                    _allSafe := FALSE;
                END_IF;
            END_IF;
        END_FOR;

        IF _allSafe THEN
            Ctrl.Status.State := ST_EXECUTE;
            Ctrl.Status.StStep := 1;
        END_IF;

        IF NOT Ctrl.Interlocks.IntlkEstop THEN
            Ctrl.Status.State     := ST_ABORTING;
            Ctrl.Faults.FaultCode := USInt#2;
            Ctrl.Faults.FltAny    := TRUE;
        END_IF;

    // ── EXECUTE ──────────────────────────────────────────────────────────────
    ST_EXECUTE:
        // ── Detect device faults ──────────────────────────────────────
        _faultSeverity := USInt#0;
        _faultDevice   := 0;
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled AND Devices[_n].Status.FltAny THEN
                IF Devices[_n].Config.FltSeverity > _faultSeverity THEN
                    _faultSeverity := Devices[_n].Config.FltSeverity;
                    _faultDevice   := _n;
                END_IF;
            END_IF;
        END_FOR;

        IF _faultSeverity = USInt#1 THEN
            // Hold — recoverable fault
            Ctrl.Status.State       := ST_HOLDING;
            Ctrl.Faults.FaultCode   := USInt#3;
            Ctrl.Faults.FaultDevice := _faultDevice;
            Ctrl.Faults.FaultStep   := Ctrl.Status.StStep;
            Ctrl.Faults.FltAny      := TRUE;
            _holdFromPause          := FALSE;
        ELSIF _faultSeverity = USInt#2 THEN
            // Abort — critical fault
            Ctrl.Status.State       := ST_ABORTING;
            Ctrl.Faults.FaultCode   := USInt#3;
            Ctrl.Faults.FaultDevice := _faultDevice;
            Ctrl.Faults.FaultStep   := Ctrl.Status.StStep;
            Ctrl.Faults.FltAny      := TRUE;
        END_IF;

        // ── Execute step (only if we're still in EXECUTE) ───────────────────
        IF Ctrl.Status.State = ST_EXECUTE THEN

            CASE Steps[Ctrl.Status.StStep].StepType OF

                // StepType 0: Action
                0:
                    FOR _n := 1 TO nDevices DO
                        IF Devices[_n].Config.Enabled THEN
                            _bitVal := SHR(IN := Steps[Ctrl.Status.StStep].ActMask,
                                           N  := INT_TO_UINT(_n - 1)) AND DWORD#1;
                            IF _bitVal = DWORD#1 THEN
                                Devices[_n].Commands.CmdAuto :=
                                    (SHR(IN := Steps[Ctrl.Status.StStep].ActTarget,
                                         N  := INT_TO_UINT(_n - 1)) AND DWORD#1) = DWORD#1;
                                IF Devices[_n].Config.DevType >= USInt#2 THEN
                                    Devices[_n].Commands.CmdSetpoint :=
                                        Steps[Ctrl.Status.StStep].ActSetpoint[_n];
                                END_IF;
                            END_IF;
                        END_IF;
                    END_FOR;

                    _stepDone := TRUE;
                    IF Steps[Ctrl.Status.StStep].CondMask <> DWORD#0 THEN
                        FOR _n := 1 TO nDevices DO
                            IF Devices[_n].Config.Enabled THEN
                                _bitVal := SHR(IN := Steps[Ctrl.Status.StStep].CondMask,
                                               N  := INT_TO_UINT(_n - 1)) AND DWORD#1;
                                IF _bitVal = DWORD#1 THEN
                                    IF (SHR(IN := Steps[Ctrl.Status.StStep].CondTarget,
                                            N  := INT_TO_UINT(_n - 1)) AND DWORD#1) = DWORD#1 THEN
                                        _stepDone := _stepDone AND Devices[_n].Status.StDone;
                                    ELSE
                                        _stepDone := _stepDone AND NOT Devices[_n].Status.StDone;
                                    END_IF;
                                END_IF;
                            END_IF;
                        END_FOR;
                    END_IF;

                // StepType 1: Wait
                1:
                    _tWait(IN := TRUE,
                           PT := DINT_TO_TIME(REAL_TO_DINT(
                                    Steps[Ctrl.Status.StStep].WaitTime * 1000.0)));
                    _stepDone := _tWait.Q;

                // StepType 2: External condition
                2:
                    IF Steps[Ctrl.Status.StStep].ExtCondIdx >= 1 AND
                       Steps[Ctrl.Status.StStep].ExtCondIdx <= 8 THEN
                        _stepDone := ExtCond[Steps[Ctrl.Status.StStep].ExtCondIdx];
                    ELSE
                        _stepDone := FALSE;
                    END_IF;

                // StepType 3: SubSequence
                3:
                    IF Steps[Ctrl.Status.StStep].SubSeqIdx >= 1 AND
                       Steps[Ctrl.Status.StStep].SubSeqIdx <= 8 THEN
                        SubStart[Steps[Ctrl.Status.StStep].SubSeqIdx] := TRUE;
                        IF Steps[Ctrl.Status.StStep].Parallel THEN
                            _stepDone := TRUE;
                        ELSE
                            _stepDone := SubDone[Steps[Ctrl.Status.StStep].SubSeqIdx];
                        END_IF;
                    ELSE
                        _stepDone := FALSE;
                    END_IF;

            END_CASE;

            // ── Step timeout ──────────────────────────────────────────
            _tStepTout(
                IN := (Steps[Ctrl.Status.StStep].Timeout > 0.0) AND NOT _stepDone,
                PT := DINT_TO_TIME(REAL_TO_DINT(
                         Steps[Ctrl.Status.StStep].Timeout * 1000.0)));
            IF _tStepTout.Q THEN
                Ctrl.Status.State     := ST_ABORTING;
                Ctrl.Faults.FaultCode := USInt#1;   // StepTimeout
                Ctrl.Faults.FaultStep := Ctrl.Status.StStep;
                Ctrl.Faults.FltAny    := TRUE;
            END_IF;

            // ── Advance step ──────────────────────────────────────────
            IF _stepDone AND Ctrl.Status.State = ST_EXECUTE THEN
                IF Steps[Ctrl.Status.StStep].StepType = 3 THEN
                    SubStart[Steps[Ctrl.Status.StStep].SubSeqIdx] := FALSE;
                END_IF;
                _tWait(IN := FALSE, PT := T#0MS);

                IF Ctrl.Status.StStep < nSteps THEN
                    Ctrl.Status.StStep := Ctrl.Status.StStep + 1;
                ELSE
                    Ctrl.Status.State := ST_COMPLETING;
                END_IF;
            END_IF;

        END_IF;

        // Command-driven transitions
        IF Ctrl.Commands.CmdPause AND NOT _prevPause THEN
            Ctrl.Status.State  := ST_PAUSING;
            _holdFromPause     := TRUE;
        END_IF;
        IF Ctrl.Commands.CmdStop AND NOT _prevStop THEN
            Ctrl.Status.State := ST_STOPPING;
        END_IF;

    // ── COMPLETING ───────────────────────────────────────────────────────────
    ST_COMPLETING:
        // Controlled shutdown: command all devices to their base state
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                Devices[_n].Commands.CmdAuto     := FALSE;
                Devices[_n].Commands.CmdSetpoint := 0.0;
            END_IF;
        END_FOR;

        // Check that all devices confirmed base state
        _allSafe := TRUE;
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled AND Devices[_n].Status.StDone THEN
                _allSafe := FALSE;
            END_IF;
        END_FOR;

        IF _allSafe THEN
            Ctrl.Status.State  := ST_COMPLETE;
            Ctrl.Status.StDone := TRUE;
        END_IF;

    // ── COMPLETE ─────────────────────────────────────────────────────────────
    ST_COMPLETE:
        IF Ctrl.Commands.CmdReset AND NOT _prevReset THEN
            Ctrl.Status.State := ST_IDLE;
        END_IF;

    // ── HOLDING ──────────────────────────────────────────────────────────────
    ST_HOLDING:
        // Command a safe state without losing StStep
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                Devices[_n].Commands.CmdAuto     := FALSE;
                Devices[_n].Commands.CmdSetpoint := 0.0;
            END_IF;
        END_FOR;

        _allSafe := TRUE;
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled AND Devices[_n].Status.StDone THEN
                _allSafe := FALSE;
            END_IF;
        END_FOR;

        // HOLDING timeout → ABORTING if it doesn't stabilize
        _tHoldTout(
            IN := (Ctrl.Config.HoldTimeout > 0.0),
            PT := DINT_TO_TIME(REAL_TO_DINT(Ctrl.Config.HoldTimeout * 1000.0)));
        IF _tHoldTout.Q THEN
            Ctrl.Status.State     := ST_ABORTING;
            Ctrl.Faults.FaultCode := USInt#4;   // HoldTimeout
            _tHoldTout(IN := FALSE, PT := T#0MS);
        END_IF;

        IF _allSafe THEN
            Ctrl.Status.State := ST_HELD;
            _tHoldTout(IN := FALSE, PT := T#0MS);
        END_IF;

    // ── HELD ─────────────────────────────────────────────────────────────────
    ST_HELD:
        // Manual resume
        IF Ctrl.Commands.CmdResume AND NOT _prevResume THEN
            Ctrl.Status.State := ST_RESUMING;
        END_IF;

        // Auto-resume: if the fault cleared and Config.AutoResume = TRUE
        IF Ctrl.Config.AutoResume THEN
            _allSafe := TRUE;
            FOR _n := 1 TO nDevices DO
                IF Devices[_n].Config.Enabled AND Devices[_n].Status.FltAny THEN
                    _allSafe := FALSE;
                END_IF;
            END_FOR;

            _tAutoResume(
                IN := _allSafe AND (Ctrl.Config.AutoResumeTime > 0.0),
                PT := DINT_TO_TIME(REAL_TO_DINT(Ctrl.Config.AutoResumeTime * 1000.0)));

            IF _allSafe AND (Ctrl.Config.AutoResumeTime = 0.0 OR _tAutoResume.Q) THEN
                Ctrl.Status.State := ST_RESUMING;
                Ctrl.Faults.FltAny    := FALSE;
                Ctrl.Faults.FaultCode := USInt#0;
                _tAutoResume(IN := FALSE, PT := T#0MS);
            END_IF;
        END_IF;

        IF Ctrl.Commands.CmdStop AND NOT _prevStop THEN
            Ctrl.Status.State := ST_STOPPING;
        END_IF;

    // ── RESUMING ─────────────────────────────────────────────────────────────
    ST_RESUMING:
        // Check conditions before resuming
        _allSafe := TRUE;
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                IF Devices[_n].Status.FltAny OR NOT Devices[_n].Status.StReady THEN
                    _allSafe := FALSE;
                END_IF;
            END_IF;
        END_FOR;

        IF _allSafe THEN
            Ctrl.Status.State     := ST_EXECUTE;
            Ctrl.Faults.FltAny    := FALSE;
            Ctrl.Faults.FaultCode := USInt#0;
        END_IF;

    // ── PAUSING ──────────────────────────────────────────────────────────────
    ST_PAUSING:
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                Devices[_n].Commands.CmdAuto     := FALSE;
                Devices[_n].Commands.CmdSetpoint := 0.0;
            END_IF;
        END_FOR;

        _allSafe := TRUE;
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled AND Devices[_n].Status.StDone THEN
                _allSafe := FALSE;
            END_IF;
        END_FOR;

        IF _allSafe THEN
            Ctrl.Status.State := ST_PAUSED;
        END_IF;

    // ── PAUSED ───────────────────────────────────────────────────────────────
    ST_PAUSED:
        IF Ctrl.Commands.CmdResume AND NOT _prevResume THEN
            Ctrl.Status.State := ST_RESUMING;
        END_IF;
        IF Ctrl.Commands.CmdStop AND NOT _prevStop THEN
            Ctrl.Status.State := ST_STOPPING;
        END_IF;

    // ── STOPPING ─────────────────────────────────────────────────────────────
    ST_STOPPING:
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                Devices[_n].Commands.CmdAuto     := FALSE;
                Devices[_n].Commands.CmdSetpoint := 0.0;
            END_IF;
        END_FOR;

        _allSafe := TRUE;
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled AND Devices[_n].Status.StDone THEN
                _allSafe := FALSE;
            END_IF;
        END_FOR;

        _tStopTout(
            IN := NOT _allSafe,
            PT := DINT_TO_TIME(REAL_TO_DINT(Ctrl.Config.StopTimeout * 1000.0)));

        IF _allSafe OR _tStopTout.Q THEN
            Ctrl.Status.State := ST_STOPPED;
            _tStopTout(IN := FALSE, PT := T#0MS);
        END_IF;

    // ── STOPPED ──────────────────────────────────────────────────────────────
    ST_STOPPED:
        IF Ctrl.Commands.CmdReset AND NOT _prevReset THEN
            Ctrl.Status.State := ST_IDLE;
        END_IF;

    // ── ABORTING ─────────────────────────────────────────────────────────────
    ST_ABORTING:
        // Emergency shutdown — more aggressive, doesn't wait for position confirmation
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                Devices[_n].Commands.CmdAuto     := FALSE;
                Devices[_n].Commands.CmdSetpoint := 0.0;
            END_IF;
        END_FOR;

        _tStopTout(
            IN := TRUE,
            PT := DINT_TO_TIME(REAL_TO_DINT(Ctrl.Config.StopTimeout * 1000.0)));

        // In ABORTING we don't wait for device confirmation — only the timeout
        IF _tStopTout.Q THEN
            Ctrl.Status.State := ST_ABORTED;
            _tStopTout(IN := FALSE, PT := T#0MS);
        END_IF;

    // ── ABORTED ──────────────────────────────────────────────────────────────
    ST_ABORTED:
        // Reset only if E-Stop OK and reset command given
        IF Ctrl.Commands.CmdReset AND NOT _prevReset
           AND Ctrl.Interlocks.IntlkEstop THEN
            Ctrl.Status.State     := ST_IDLE;
            Ctrl.Faults.FltAny    := FALSE;
            Ctrl.Faults.FaultCode := USInt#0;
        END_IF;

END_CASE;

// ── Status outputs ────────────────────────────────────────────────────────────
Ctrl.Status.StIdle       := (Ctrl.Status.State = ST_IDLE);
Ctrl.Status.StStarting   := (Ctrl.Status.State = ST_STARTING);
Ctrl.Status.StExecute    := (Ctrl.Status.State = ST_EXECUTE);
Ctrl.Status.StCompleting := (Ctrl.Status.State = ST_COMPLETING);
Ctrl.Status.StComplete   := (Ctrl.Status.State = ST_COMPLETE);
Ctrl.Status.StHolding    := (Ctrl.Status.State = ST_HOLDING);
Ctrl.Status.StHeld       := (Ctrl.Status.State = ST_HELD);
Ctrl.Status.StResuming   := (Ctrl.Status.State = ST_RESUMING);
Ctrl.Status.StPausing    := (Ctrl.Status.State = ST_PAUSING);
Ctrl.Status.StPaused     := (Ctrl.Status.State = ST_PAUSED);
Ctrl.Status.StStopping   := (Ctrl.Status.State = ST_STOPPING);
Ctrl.Status.StStopped    := (Ctrl.Status.State = ST_STOPPED);
Ctrl.Status.StAborting   := (Ctrl.Status.State = ST_ABORTING);
Ctrl.Status.StAborted    := (Ctrl.Status.State = ST_ABORTED);

// ── Edge-detection memory ────────────────────────────────────────────────────
_prevStart  := Ctrl.Commands.CmdStart;
_prevStop   := Ctrl.Commands.CmdStop;
_prevPause  := Ctrl.Commands.CmdPause;
_prevResume := Ctrl.Commands.CmdResume;
_prevAbort  := Ctrl.Commands.CmdAbort;
_prevReset  := Ctrl.Commands.CmdReset;

END_FUNCTION_BLOCK
```

---

## 6. FB_SeqMaster

**Suggested number:** FB210

```scl
FUNCTION_BLOCK "FB_SeqMaster"

VAR_INPUT
    nSubSeqs    : Int;
END_VAR

VAR_IN_OUT
    Ctrl        : "UDT_MasterCtrl";
    SubCtrl     : ARRAY[1..8] OF "UDT_SeqCtrl";    // Ctrl of each Runner
END_VAR

VAR
    _prevStart  : Bool;
    _prevStop   : Bool;
    _prevPause  : Bool;
    _prevResume : Bool;
    _prevAbort  : Bool;
    _prevReset  : Bool;
    _allDone    : Bool;
    _allSafe    : Bool;
END_VAR

VAR_TEMP
    _n          : Int;
    _faultSub   : USInt;
    _policy     : USInt;
END_VAR

VAR CONSTANT
    ST_IDLE     : USInt := 0;
    ST_EXECUTE  : USInt := 2;
    ST_HELD     : USInt := 6;
    ST_PAUSED   : USInt := 9;
    ST_STOPPING : USInt := 11;
    ST_STOPPED  : USInt := 12;
    ST_ABORTING : USInt := 13;
    ST_ABORTED  : USInt := 14;
END_VAR

// ─────────────────────────────────────────────────────────────────────────────
// BODY
// ─────────────────────────────────────────────────────────────────────────────

// ── Global E-Stop and CmdAbort ────────────────────────────────────────────────
IF (Ctrl.Commands.CmdAbort AND NOT _prevAbort)
   OR NOT Ctrl.Interlocks.IntlkEstop THEN
    Ctrl.Status.State := ST_ABORTING;
    FOR _n := 1 TO nSubSeqs DO
        SubCtrl[_n].Commands.CmdAbort := TRUE;
    END_FOR;
END_IF;

// ── Detect sub-sequence faults ─────────────────────────────────────────
FOR _n := 1 TO nSubSeqs DO
    IF SubCtrl[_n].Status.StAborted OR SubCtrl[_n].Status.StHeld THEN
        _faultSub := INT_TO_USINT(_n);

        IF Ctrl.Config.SubPolicy[_n].IsMandatory THEN
            _policy := Ctrl.Config.SubPolicy[_n].OnFaultPolicy;

            CASE _policy OF
                0:  // AbortAll
                    Ctrl.Status.State           := ST_ABORTING;
                    Ctrl.Faults.FaultSubSeq     := _faultSub;
                    Ctrl.Faults.FltAny          := TRUE;
                    FOR _n := 1 TO nSubSeqs DO
                        SubCtrl[_n].Commands.CmdAbort := TRUE;
                    END_FOR;
                1:  // AbortOnly — only the failed runner, the others continue
                    Ctrl.Faults.FaultSubSeq     := _faultSub;
                    Ctrl.Faults.FltAny          := TRUE;
                2:  // HoldAll
                    Ctrl.Status.State           := ST_HELD;
                    Ctrl.Faults.FaultSubSeq     := _faultSub;
                    Ctrl.Faults.FltAny          := TRUE;
                    FOR _n := 1 TO nSubSeqs DO
                        SubCtrl[_n].Commands.CmdStop := TRUE;
                    END_FOR;
                3:  // HoldOnly — only the failed runner
                    Ctrl.Faults.FaultSubSeq     := _faultSub;
                    Ctrl.Faults.FltAny          := TRUE;
            END_CASE;
        END_IF;
    END_IF;
END_FOR;

// ── Master state machine ─────────────────────────────────────────────────
CASE Ctrl.Status.State OF

    ST_IDLE:
        IF Ctrl.Commands.CmdStart AND NOT _prevStart
           AND Ctrl.Interlocks.IntlkReady
           AND Ctrl.Interlocks.IntlkEstop THEN
            Ctrl.Status.State  := ST_EXECUTE;
            Ctrl.Status.StDone := FALSE;
            // Launch sub-sequences per configuration
            FOR _n := 1 TO nSubSeqs DO
                SubCtrl[_n].Commands.CmdStart    := TRUE;
                SubCtrl[_n].Interlocks.IntlkReady  := TRUE;
                SubCtrl[_n].Interlocks.IntlkEstop  := Ctrl.Interlocks.IntlkEstop;
            END_FOR;
        END_IF;

    ST_EXECUTE:
        // Propagate E-Stop to sub-sequences
        FOR _n := 1 TO nSubSeqs DO
            SubCtrl[_n].Interlocks.IntlkEstop := Ctrl.Interlocks.IntlkEstop;
        END_FOR;

        // Check whether all sub-sequences have completed
        _allDone := TRUE;
        FOR _n := 1 TO nSubSeqs DO
            IF NOT SubCtrl[_n].Status.StComplete
               AND NOT SubCtrl[_n].Status.StIdle THEN
                _allDone := FALSE;
            END_IF;
        END_FOR;
        IF _allDone THEN
            Ctrl.Status.State  := ST_IDLE;
            Ctrl.Status.StDone := TRUE;
        END_IF;

        IF Ctrl.Commands.CmdPause AND NOT _prevPause THEN
            Ctrl.Status.State := ST_PAUSED;
            FOR _n := 1 TO nSubSeqs DO
                SubCtrl[_n].Commands.CmdPause := TRUE;
            END_FOR;
        END_IF;

        IF Ctrl.Commands.CmdStop AND NOT _prevStop THEN
            Ctrl.Status.State := ST_STOPPING;
            FOR _n := 1 TO nSubSeqs DO
                SubCtrl[_n].Commands.CmdStop := TRUE;
            END_FOR;
        END_IF;

    ST_HELD:
        IF Ctrl.Commands.CmdResume AND NOT _prevResume THEN
            Ctrl.Status.State := ST_EXECUTE;
            FOR _n := 1 TO nSubSeqs DO
                SubCtrl[_n].Commands.CmdResume := TRUE;
            END_FOR;
            Ctrl.Faults.FltAny    := FALSE;
            Ctrl.Faults.FaultCode := USInt#0;
        END_IF;

    ST_PAUSED:
        IF Ctrl.Commands.CmdResume AND NOT _prevResume THEN
            Ctrl.Status.State := ST_EXECUTE;
            FOR _n := 1 TO nSubSeqs DO
                SubCtrl[_n].Commands.CmdResume := TRUE;
            END_FOR;
        END_IF;
        IF Ctrl.Commands.CmdStop AND NOT _prevStop THEN
            Ctrl.Status.State := ST_STOPPING;
            FOR _n := 1 TO nSubSeqs DO
                SubCtrl[_n].Commands.CmdStop := TRUE;
            END_FOR;
        END_IF;

    ST_STOPPING:
        _allSafe := TRUE;
        FOR _n := 1 TO nSubSeqs DO
            IF NOT SubCtrl[_n].Status.StStopped
               AND NOT SubCtrl[_n].Status.StIdle THEN
                _allSafe := FALSE;
            END_IF;
        END_FOR;
        IF _allSafe THEN
            Ctrl.Status.State := ST_STOPPED;
        END_IF;

    ST_STOPPED:
        IF Ctrl.Commands.CmdReset AND NOT _prevReset THEN
            Ctrl.Status.State := ST_IDLE;
            FOR _n := 1 TO nSubSeqs DO
                SubCtrl[_n].Commands.CmdReset := TRUE;
            END_FOR;
        END_IF;

    ST_ABORTING:
        _allSafe := TRUE;
        FOR _n := 1 TO nSubSeqs DO
            IF NOT SubCtrl[_n].Status.StAborted
               AND NOT SubCtrl[_n].Status.StIdle THEN
                _allSafe := FALSE;
            END_IF;
        END_FOR;
        IF _allSafe THEN
            Ctrl.Status.State := ST_ABORTED;
        END_IF;

    ST_ABORTED:
        IF Ctrl.Commands.CmdReset AND NOT _prevReset
           AND Ctrl.Interlocks.IntlkEstop THEN
            Ctrl.Status.State     := ST_IDLE;
            Ctrl.Faults.FltAny    := FALSE;
            Ctrl.Faults.FaultCode := USInt#0;
            FOR _n := 1 TO nSubSeqs DO
                SubCtrl[_n].Commands.CmdReset := TRUE;
            END_FOR;
        END_IF;

END_CASE;

// ── Status outputs ─────────────────────────────────────────────────────────────
Ctrl.Status.StIdle    := (Ctrl.Status.State = ST_IDLE);
Ctrl.Status.StExecute := (Ctrl.Status.State = ST_EXECUTE);
Ctrl.Status.StHeld    := (Ctrl.Status.State = ST_HELD);
Ctrl.Status.StPaused  := (Ctrl.Status.State = ST_PAUSED);
Ctrl.Status.StStopped := (Ctrl.Status.State = ST_STOPPED);
Ctrl.Status.StAborted := (Ctrl.Status.State = ST_ABORTED);

// ── Edge-detection memory ─────────────────────────────────────────────────────
_prevStart  := Ctrl.Commands.CmdStart;
_prevStop   := Ctrl.Commands.CmdStop;
_prevPause  := Ctrl.Commands.CmdPause;
_prevResume := Ctrl.Commands.CmdResume;
_prevAbort  := Ctrl.Commands.CmdAbort;
_prevReset  := Ctrl.Commands.CmdReset;

END_FUNCTION_BLOCK
```

---

## 7. How to set it up in TIA Portal

### Creation order

```
1. UDT_MasterSubConfig
2. UDT_DeviceCtrl
3. UDT_SeqStep
4. UDT_SeqCtrl
5. UDT_MasterCtrl
6. DB_DEVICES          (optimized OFF if using PUT/GET)
7. DB_SEQUENCES
8. FB_SeqRunner [FB211]
9. FB_SeqMaster [FB210]
```

### Initialization in OB100

```scl
// Configure runners
"DB_SEQUENCES".Sub1Ctrl.Config.StopTimeout   := 5.0;
"DB_SEQUENCES".Sub1Ctrl.Config.HoldTimeout   := 30.0;
"DB_SEQUENCES".Sub1Ctrl.Config.AutoResume    := FALSE;
"DB_SEQUENCES".Sub1Ctrl.Config.AutoResumeTime := 0.0;

// Configure the master's policy per sub-sequence
"DB_SEQUENCES".MainCtrl.Config.SubPolicy[1].OnFaultPolicy   := USInt#2;  // HoldAll
"DB_SEQUENCES".MainCtrl.Config.SubPolicy[1].OnTimeoutPolicy := USInt#0;  // AbortAll
"DB_SEQUENCES".MainCtrl.Config.SubPolicy[1].IsMandatory     := TRUE;

// Configure devices
"DB_DEVICES".Valves[1].Config.DevType     := USInt#0;
"DB_DEVICES".Valves[1].Config.Enabled     := TRUE;
"DB_DEVICES".Valves[1].Config.FltSeverity := USInt#1;  // Hold
```

### Connecting to existing device FBs

```scl
// The device FB reads Commands and writes Status — a single IN_OUT
"FB_Valve".Valve_01(
    CmdAuto  := "DB_DEVICES".Valves[1].Commands.CmdAuto,
    StDone   => "DB_DEVICES".Valves[1].Status.StDone,
    StReady  => "DB_DEVICES".Valves[1].Status.StReady,
    FltAny   => "DB_DEVICES".Valves[1].Status.FltAny,
    Enabled  := "DB_DEVICES".Valves[1].Config.Enabled);
```

---

## 8. How to use it — examples

### Example 1: Valve routing — 5 mutually exclusive routings

```scl
// In FB_FillingMaster, routing-exclusion logic:
// CmdStart is only sent to the requested routing's runner.
// Controlled handoff: wait for the active one's StStopped before starting the new one.

IF #RoutingRequest <> #RoutingActive THEN
    // Step 1: stop the active routing
    "DB_SEQUENCES".Sub1Ctrl.Commands.CmdStop := (#RoutingActive = 1);
    "DB_SEQUENCES".Sub2Ctrl.Commands.CmdStop := (#RoutingActive = 2);
    // ... etc

    // Step 2: once the active one confirms it stopped, start the new one
    IF "DB_SEQUENCES".Sub1Ctrl.Status.StStopped OR
       "DB_SEQUENCES".Sub1Ctrl.Status.StIdle THEN
        #RoutingActive := #RoutingRequest;
        "DB_SEQUENCES".Sub1Ctrl.Commands.CmdStart := (#RoutingRequest = 1);
        "DB_SEQUENCES".Sub2Ctrl.Commands.CmdStart := (#RoutingRequest = 2);
    END_IF;
END_IF;
```

### Example 2: Checking state for machine logic

```scl
// Operation permissive only when the sequence is in EXECUTE and step >= 3
IF "DB_SEQUENCES".Sub1Ctrl.Status.StExecute AND
   "DB_SEQUENCES".Sub1Ctrl.Status.StStep >= 3 THEN
    "Outputs".PumpPermit := TRUE;
END_IF;

// Alarm the manager if the runner is in ABORTED
IF "DB_SEQUENCES".Sub1Ctrl.Status.StAborted THEN
    #AlarmSeqAborted    := TRUE;
    #AlarmFaultCode     := "DB_SEQUENCES".Sub1Ctrl.Faults.FaultCode;
    #AlarmFaultStep     := "DB_SEQUENCES".Sub1Ctrl.Faults.FaultStep;
    #AlarmFaultDevice   := "DB_SEQUENCES".Sub1Ctrl.Faults.FaultDevice;
END_IF;
```

### Example 3: Auto-resume for a transient valve fault

```scl
// In OB100: configure the runner for auto-resume with a 5-second confirmation window
"DB_SEQUENCES".Sub1Ctrl.Config.AutoResume     := TRUE;
"DB_SEQUENCES".Sub1Ctrl.Config.AutoResumeTime := 5.0;

// A valve with FltSeverity=1 (Hold): if it loses feedback and recovers it in < 5s,
// the runner automatically returns to EXECUTE from the same step.
// If it doesn't recover, it stays in HELD until a manual CmdResume.
```

---

## 9. Scaling and topologies

### Topology A — Simple machine (standalone Runner)

```
OB1 → FB_SeqRunner → DB_DEVICES.Valves[1..8]
```
No Master. Commands go straight from the HMI to `DB_SEQUENCES.Sub1Ctrl.Commands`.

### Topology B — Mutually exclusive routings (Master as arbiter)

```
FB_SeqMaster
  ├── FB_SeqRunner [Routing_A]  ← only one active at a time
  ├── FB_SeqRunner [Routing_B]
  └── FB_SeqRunner [Routing_C]
```
The Master doesn't run steps — it only manages exclusion and handoff between runners.

### Topology C — Complex process with parallel branches

```
FB_SeqMaster "Production"
  ├── FB_SeqRunner [Fill]      → parallel
  ├── FB_SeqRunner [Heat]      → parallel
  └── FB_SeqRunner [Agitate]   → parallel, IsMandatory=FALSE
```
All three run simultaneously. Agitate can fail without aborting production.

### Topology D — Hierarchical, multi-area

```
FB_SeqMaster "Plant"
  ├── FB_SeqMaster "Area_Filling"
  │    ├── FB_SeqRunner [...]
  │    └── FB_SeqRunner [...]
  └── FB_SeqMaster "Area_CIP"
       ├── FB_SeqRunner [...]
       └── FB_SeqRunner [...]
```
Cross-interlocks between area masters:
```scl
"DB_SEQ_CIP".MainCtrl.Interlocks.IntlkReady :=
    "DB_SEQ_FILLING".MainCtrl.Status.StIdle;  // CIP only if Filling is idle
```

---

## 10. Notes and limitations

### Commands on the Master: pulses vs. levels
The commands (`CmdStart`, `CmdStop`, etc.) in `UDT_SeqCtrl.Commands` are level Bools.
The Runner detects the edge internally. The HMI can write either a short pulse or a level —
both work. Recommended: use pulses from the HMI to avoid the command getting "stuck."

### Master → Sub command propagation
The Master writes directly into `SubCtrl[n].Commands`. This means the Master
and the operator could compete if the HMI also writes to those fields. Recommended
convention: the HMI writes to `MasterCtrl.Commands`, and the Master propagates to the runners.
Don't allow the HMI to write directly to `SubCtrl`.

### CmdClear — not implemented in this version
PackML defines `CmdClear` to clear faults without changing state. Suggested implementation:
in the HELD or ABORTED state, `CmdClear` clears `Faults.*` without a state transition.
Add as an extension if the HMI requires it.

### REAL_TO_TIME on S7-1200
Always use `DINT_TO_TIME(REAL_TO_DINT(value * 1000.0))` as shown in the code.
`REAL_TO_TIME` directly is not available on all S7-1200 firmware versions.

### 32-device limit per Runner
Determined by the width of the `DWORD` in ActMask/CondMask.
For S7-1500: replace `DWORD` with `LWORD` and adjust the SHR calls to extend to 64 devices.
