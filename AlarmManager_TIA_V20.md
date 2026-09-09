# Alarm Manager — TIA Portal V20
**Architecture: FB_AlarmHandler + FC_AlarmManager**
Version 1.0

---

## Table of contents
1. [General architecture](#1-general-architecture)
2. [UDTs](#2-udts)
3. [Global DBs](#3-global-dbs)
4. [FB_AlarmHandler](#4-fb_alarmhandler)
5. [FC_AlarmManager](#5-fc_alarmmanager)
6. [How to set it up in TIA Portal](#6-how-to-set-it-up-in-tia-portal)
7. [How to use it — examples](#7-how-to-use-it--examples)
8. [Scaling to large projects](#8-scaling-to-large-projects)
9. [Notes and limitations](#9-notes-and-limitations)

---

## 1. General architecture

```
OB1 (every cycle)
 └── FC_AlarmManager
      ├── FB_AlarmHandler (instance Motor_01)  ──► DB_ALARMS.Motors[0]  ──► DB_HIST
      ├── FB_AlarmHandler (instance Motor_02)  ──► DB_ALARMS.Motors[1]  ──► DB_HIST
      ├── FB_AlarmHandler (instance Sensor_01) ──► DB_ALARMS.Sensors[0] ──► DB_HIST
      ├── ...
      └── CmdAckAll / CmdClear (processed at the end of the FC)
```

**Design principles:**
- **O(1) access** to each alarm's state — no linear searches.
- **One FB instance per alarm** — each one has its own edge-detection state.
- **Optional history** — disabled with `EnableHistory := FALSE` for CPUs without an HMI or with limited resources.
- **Two separate layers**: active state (used by PLC logic) and history (read by HMI over OPC-UA/PUT-GET).

---

## 2. UDTs

Create in TIA Portal in the order shown (there are dependencies between UDTs).

### UDT_AlarmState
> Active state of a single alarm. Lives in `DB_ALARMS`.

```scl
TYPE "UDT_AlarmState"
    STRUCT
        AlarmID     : DInt;         // Unique alarm identifier
        Active      : Bool;         // TRUE = alarm currently active
        Acknowledged: Bool;         // TRUE = operator acknowledged the alarm
        TimeRaised  : DTL;          // Timestamp of (most recent) activation
        TimeCleared : DTL;          // Timestamp of (most recent) deactivation
        Count       : Int;          // Number of times it has activated
        Severity    : USInt;        // 0=Info  1=Warning  2=Error  3=Critical
        Category    : USInt;        // Category: 1=Motor 2=Sensor 3=Safety etc.
        Device      : USInt;        // Device ID within the category
    END_STRUCT;
END_TYPE
```

### UDT_AlarmHistEntry
> One event record in the history. Lives in `DB_HIST`.
> No `Message` field — the HMI maps text using `AlarmID` or the `(Severity, Category, Device)` combination.

```scl
TYPE "UDT_AlarmHistEntry"
    STRUCT
        AlarmID     : DInt;
        Severity    : USInt;
        Category    : USInt;
        Device      : USInt;
        TimeRaised  : DTL;
        TimeCleared : DTL;
        Acknowledged: Bool;
        Valid       : Bool;         // FALSE = empty entry (free slot in the ring buffer)
    END_STRUCT;
END_TYPE
```

### UDT_AlarmHistBuffer
> History ring buffer. Size configurable per project.

```scl
TYPE "UDT_AlarmHistBuffer"
    STRUCT
        Head        : Int;          // Next write index
        Tail        : Int;          // Next read index (external consumption)
        Count       : Int;          // Currently valid entries in the buffer
        MaxSize     : Int;          // Maximum size (same as the declared array)
        Full        : Bool;         // TRUE = buffer full, next write overwrites
        Entries     : Array[0..99] of "UDT_AlarmHistEntry";  // Adjust size per project
    END_STRUCT;
END_TYPE
```

> **Note:** If you need more or less history, change the `Entries` array to `Array[0..N-1]`
> and set `MaxSize` to the same value N in `DB_HIST`. The FB's code doesn't change.

---

## 3. Global DBs

### DB_ALARMS — Active state by category

```scl
DATA_BLOCK "DB_ALARMS"
    VAR
        Motors   : Array[0..99] of "UDT_AlarmState";   // IDs 0..99
        Sensors  : Array[0..99] of "UDT_AlarmState";   // IDs 100..199 (convention)
        Safety   : Array[0..49]  of "UDT_AlarmState";  // IDs 200..249
        General  : Array[0..49]  of "UDT_AlarmState";  // IDs 250..299
        // Add categories as needed per project
    END_VAR
END_DATA_BLOCK
```

> Each project only defines the categories it needs.
> The array index is the index within the category, not the absolute AlarmID.

### DB_HIST — Shared history

```scl
DATA_BLOCK "DB_HIST"
    VAR
        Buf : "UDT_AlarmHistBuffer";
    END_VAR
END_DATA_BLOCK
```

> Initialize `DB_HIST`.Buf.MaxSize := 100 on the first scan (OB100 or init logic).

---

## 4. FB_AlarmHandler

**Suggested number:** FB100
**Function:** Manages the complete lifecycle of a single alarm.

```scl
FUNCTION_BLOCK "FB_AlarmHandler"

// ---------------------------------------------------------------------------
// INTERFACE
// ---------------------------------------------------------------------------
VAR_INPUT
    IsActive        : Bool;         // Alarm signal (from the process)
    AlarmID         : DInt;         // Unique ID for this alarm
    Category        : USInt;        // Category (for history)
    Severity        : USInt;        // Severity (for history)
    Device          : USInt;        // Device (for history)
    CmdAck          : Bool;         // Individual ACK for this alarm
    EnableHistory   : Bool;         // FALSE = don't write to history (saves cycle time)
END_VAR

VAR_IN_OUT
    State           : "UDT_AlarmState";         // Entry in DB_ALARMS (direct read/write)
    HistBuf         : "UDT_AlarmHistBuffer";    // Shared ring buffer in DB_HIST
END_VAR

VAR                                             // Static — persist between cycles
    _PrevActive     : Bool;
    _PrevAck        : Bool;
    _HistIdx        : Int;                      // This alarm's index in the history (for TimeCleared)
    _InHistory      : Bool;                     // TRUE = there is an open entry in the history
END_VAR

VAR_TEMP
    _sysTime        : DTL;
    _i              : Int;
END_VAR

// ---------------------------------------------------------------------------
// BODY
// ---------------------------------------------------------------------------

// ── 1. RISING EDGE: IsActive ────────────────────────────────────────────────
IF IsActive AND NOT _PrevActive THEN

    // Update active state (O(1) — the caller already indexed correctly)
    State.Active        := TRUE;
    State.Acknowledged  := FALSE;
    State.AlarmID       := AlarmID;
    State.Severity      := Severity;
    State.Category      := Category;
    State.Device        := Device;
    State.Count         := State.Count + 1;

    RD_SYS_T(OUT => _sysTime);
    State.TimeRaised    := _sysTime;
    State.TimeCleared   := DTL#1970-01-01-00:00:00;    // Reset cleared timestamp

    // ── Write to history (if enabled) ──
    IF EnableHistory THEN
        // Save the index we're about to write so TimeCleared can be updated later
        _HistIdx := HistBuf.Head;
        _InHistory := TRUE;

        // Write entry
        HistBuf.Entries[HistBuf.Head].AlarmID       := AlarmID;
        HistBuf.Entries[HistBuf.Head].Severity      := Severity;
        HistBuf.Entries[HistBuf.Head].Category      := Category;
        HistBuf.Entries[HistBuf.Head].Device        := Device;
        HistBuf.Entries[HistBuf.Head].TimeRaised    := _sysTime;
        HistBuf.Entries[HistBuf.Head].TimeCleared   := DTL#1970-01-01-00:00:00;
        HistBuf.Entries[HistBuf.Head].Acknowledged  := FALSE;
        HistBuf.Entries[HistBuf.Head].Valid         := TRUE;

        // Detect a full buffer before advancing Head
        IF HistBuf.Count >= HistBuf.MaxSize THEN
            HistBuf.Full := TRUE;
            // Ring buffer full: advance Tail (discard the oldest event)
            HistBuf.Tail := (HistBuf.Tail + 1) MOD HistBuf.MaxSize;
        ELSE
            HistBuf.Count := HistBuf.Count + 1;
            HistBuf.Full  := FALSE;
        END_IF;

        // Advance Head
        HistBuf.Head := (HistBuf.Head + 1) MOD HistBuf.MaxSize;
    END_IF;

END_IF;

// ── 2. FALLING EDGE: IsActive ───────────────────────────────────────────────
IF NOT IsActive AND _PrevActive THEN

    State.Active := FALSE;

    RD_SYS_T(OUT => _sysTime);
    State.TimeCleared := _sysTime;

    // Update TimeCleared on the history entry
    IF EnableHistory AND _InHistory THEN
        HistBuf.Entries[_HistIdx].TimeCleared := _sysTime;
        _InHistory := FALSE;
    END_IF;

END_IF;

// ── 3. Individual ACK ───────────────────────────────────────────────────────
IF CmdAck AND NOT _PrevAck THEN
    State.Acknowledged := TRUE;

    // Update Acknowledged on the history entry
    IF EnableHistory AND _InHistory THEN
        HistBuf.Entries[_HistIdx].Acknowledged := TRUE;
    END_IF;
END_IF;

// ── 4. Update edge-detection memory ─────────────────────────────────────────
_PrevActive := IsActive;
_PrevAck    := CmdAck;

END_FUNCTION_BLOCK
```

---

## 5. FC_AlarmManager

**Suggested number:** FC100
**Function:** Wrapper that calls every `FB_AlarmHandler` instance and processes global commands.

```scl
FUNCTION "FC_AlarmManager" : Void

// ---------------------------------------------------------------------------
// INTERFACE
// ---------------------------------------------------------------------------
VAR_INPUT
    CmdAckAll       : Bool;         // ACK all active alarms
    CmdClear        : Bool;         // Clear the entire history
    EnableHistory   : Bool;         // Enable/disable history globally
END_VAR

VAR_TEMP
    _i              : Int;
    _sysTime        : DTL;
    _PrevAckAll     : Bool;         // NOTE: see note below (*)
    _PrevClear      : Bool;
END_VAR

// ---------------------------------------------------------------------------
// BODY
// ---------------------------------------------------------------------------

// ── CALLS TO FB_AlarmHandler INSTANCES ─────────────────────────────
// One line per alarm. The array index IS the direct O(1) access.
// Pattern: FB_AlarmHandler.InstanceName(
//             IsActive       := <process signal>,
//             AlarmID        := <constant>,
//             Category       := <constant>,
//             Severity       := <constant>,
//             Device         := <constant>,
//             CmdAck         := <individual ACK bit>,
//             EnableHistory  := EnableHistory,
//             State          := "DB_ALARMS".<Category>[<index>],
//             HistBuf        := "DB_HIST".Buf);

"FB_AlarmHandler".Motor_OverTemp(
    IsActive        := "DB_PROCESS".Motor1.TempHigh,
    AlarmID         := DInt#1001,
    Category        := USInt#1,
    Severity        := USInt#2,
    Device          := USInt#1,
    CmdAck          := "DB_CMDS".AckMotor1,
    EnableHistory   := EnableHistory,
    State           := "DB_ALARMS".Motors[0],
    HistBuf         := "DB_HIST".Buf);

"FB_AlarmHandler".Motor_Overload(
    IsActive        := "DB_PROCESS".Motor1.Overload,
    AlarmID         := DInt#1002,
    Category        := USInt#1,
    Severity        := USInt#3,
    Device          := USInt#1,
    CmdAck          := "DB_CMDS".AckMotor1_OL,
    EnableHistory   := EnableHistory,
    State           := "DB_ALARMS".Motors[1],
    HistBuf         := "DB_HIST".Buf);

"FB_AlarmHandler".Sensor_Pressure_High(
    IsActive        := "DB_PROCESS".Sensor1.PressHigh,
    AlarmID         := DInt#2001,
    Category        := USInt#2,
    Severity        := USInt#2,
    Device          := USInt#1,
    CmdAck          := "DB_CMDS".AckSensor1,
    EnableHistory   := EnableHistory,
    State           := "DB_ALARMS".Sensors[0],
    HistBuf         := "DB_HIST".Buf);

// ... add instances as needed per project ...


// ── CmdAckAll — Edge detected here with a static VAR in the caller ──────
// (*) NOTE: an FC has no static VARs. There are two options:
//
//   OPTION A (recommended): move CmdAckAll and CmdClear to an FB wrapper instead of an FC.
//   OPTION B: use an M memory bit or an auxiliary DB for the edge-detection memory.
//
// Example using Option B (bit M0.0 = AckAll edge memory, M0.1 = Clear edge memory):

IF CmdAckAll AND NOT %M0.0 THEN
    FOR _i := 0 TO 99 DO
        "DB_ALARMS".Motors[_i].Acknowledged  := TRUE;
        "DB_ALARMS".Sensors[_i].Acknowledged := TRUE;
    END_FOR;
    FOR _i := 0 TO 49 DO
        "DB_ALARMS".Safety[_i].Acknowledged  := TRUE;
        "DB_ALARMS".General[_i].Acknowledged := TRUE;
    END_FOR;
    // Also mark the history
    IF EnableHistory THEN
        FOR _i := 0 TO ("DB_HIST".Buf.MaxSize - 1) DO
            "DB_HIST".Buf.Entries[_i].Acknowledged := TRUE;
        END_FOR;
    END_IF;
END_IF;
%M0.0 := CmdAckAll;

// ── CmdClear — Clear history ─────────────────────────────────────────
IF CmdClear AND NOT %M0.1 THEN
    "DB_HIST".Buf.Head  := 0;
    "DB_HIST".Buf.Tail  := 0;
    "DB_HIST".Buf.Count := 0;
    "DB_HIST".Buf.Full  := FALSE;
    FOR _i := 0 TO ("DB_HIST".Buf.MaxSize - 1) DO
        "DB_HIST".Buf.Entries[_i].Valid := FALSE;
    END_FOR;
END_IF;
%M0.1 := CmdClear;

END_FUNCTION
```

> **Recommendation:** If CmdAckAll and CmdClear come from the HMI as short pulses, convert
> `FC_AlarmManager` into `FB_AlarmManager` to have its own static VARs and avoid
> relying on global memory bits.

---

## 6. How to set it up in TIA Portal

### Creation order

```
1. UDT_AlarmState          (PLC types > Add new data type)
2. UDT_AlarmHistEntry
3. UDT_AlarmHistBuffer
4. DB_ALARMS               (Global DB — no optimized access if using PUT/GET)
5. DB_HIST                 (Global DB — no optimized access if using PUT/GET)
6. FB_AlarmHandler  [FB100]
7. FC_AlarmManager  [FC100]
```

### DB configuration for OPC-UA / PUT-GET

If the HMI accesses the history over **OPC-UA** (S7-1200 FW4+):
- `DB_HIST`: Optimized block access = **ON** ✅ (OPC-UA supports it and it's more efficient)

If using **PUT/GET** (classic Siemens HMI, external access):
- `DB_HIST`: Optimized block access = **OFF** ⚠️ (PUT/GET requirement)
- In CPU properties: enable "Permit access with PUT/GET"

### Multiple FB instances

In TIA V15+, you can declare instances as **multi-instance** inside the parent FC/FB,
or as **individual instance DBs**. For large projects, multi-instance is cleaner.

### Initialization (OB100 — Startup)

```scl
// In OB100, initialize the history's MaxSize
"DB_HIST".Buf.MaxSize := 100;  // Must match the size of the Entries array
"DB_HIST".Buf.Head    := 0;
"DB_HIST".Buf.Tail    := 0;
"DB_HIST".Buf.Count   := 0;
"DB_HIST".Buf.Full    := FALSE;
```

---

## 7. How to use it — examples

### Example 1: Simple machine without an HMI (S7-1214, standalone)

```scl
// In FC_AlarmManager, all instances with EnableHistory := FALSE
"FB_AlarmHandler".PressureHigh(
    IsActive       := "Process".Pressure > 10.0,
    AlarmID        := DInt#1,
    Category       := USInt#1,
    Severity       := USInt#2,
    Device         := USInt#1,
    CmdAck         := "HMI".AckBtn,
    EnableHistory  := FALSE,          // No history — saves cycle time
    State          := "DB_ALARMS".General[0],
    HistBuf        := "DB_HIST".Buf);
```

### Example 2: Reading active state in machine logic

```scl
// The PLC queries the state directly — O(1) access, no need to call the FB
IF "DB_ALARMS".Motors[0].Active AND
   "DB_ALARMS".Motors[0].Severity >= USInt#2 THEN
    "Outputs".EmergencyStop := TRUE;
END_IF;

// Check whether all motor alarms are acknowledged
IF NOT "DB_ALARMS".Motors[0].Active AND
   NOT "DB_ALARMS".Motors[1].Active THEN
    "Outputs".MotorRunPermit := TRUE;
END_IF;
```

### Example 3: HMI reading history over OPC-UA

The HMI reads `DB_HIST.Buf.Entries[0..N]` and only displays entries where `Valid = TRUE`.
For the message text, the HMI keeps an internal lookup table:

```
AlarmID 1001 → "Motor 1 — High temperature"
AlarmID 1002 → "Motor 1 — Overload"
AlarmID 2001 → "Sensor 1 — High pressure"
```

Or alternatively, use `(Category, Device, Severity)` to compose the text dynamically.

### Example 4: Adding a new alarm to the system

1. Add the process signal to the process DB.
2. Add a line in `FC_AlarmManager`:

```scl
"FB_AlarmHandler".NewAlarmName(
    IsActive       := "DB_PROCESS".NewSignal,
    AlarmID        := DInt#3001,       // Unique ID, don't reuse
    Category       := USInt#3,         // Safety
    Severity       := USInt#3,         // Critical
    Device         := USInt#5,
    CmdAck         := "DB_CMDS".AckNew,
    EnableHistory  := EnableHistory,
    State          := "DB_ALARMS".Safety[0],  // free index
    HistBuf        := "DB_HIST".Buf);
```

3. Add the text to the HMI's lookup table. Done.

---

## 8. Scaling to large projects

| Scale | Recommended configuration |
|--------|--------------------------|
| ~50 alarms (simple machine) | 4 categories × 15 entries, 50-entry history, `EnableHistory := FALSE` if there's no HMI |
| ~200 alarms (production line) | 6–8 categories, 200–500-entry history, OPC-UA active |
| 500+ alarms (plant) | Consider multiple `FC_AlarmManager` per zone/area, one `DB_HIST` per area or one centralized global |
| 1000+ alarms | Split into FCs by zone. The FB doesn't change. Just add instances and categories in `DB_ALARMS` |

**Estimated memory per alarm:**
- `UDT_AlarmState`: ~50 bytes per entry
- `UDT_AlarmHistEntry`: ~40 bytes per entry
- 500 active alarms + 1000 history entries ≈ **65 KB** — well within any S7-1500.

---

## 9. Notes and limitations

### Limitation: `_HistIdx` on rapid activations
If an alarm activates/deactivates several times before the HMI reads the history,
`_HistIdx` always points to the **last** entry written. Earlier entries for that
alarm are left with `TimeCleared = 1970` until the ring buffer overwrites them.
For systems with very high-frequency alarms, consider not updating `TimeCleared`
in the history and handling it only in `State.TimeCleared`.

### Limitation: FC has no static VARs
The edge detection for `CmdAckAll` and `CmdClear` in the FC uses memory bits (`%M`).
If that's not acceptable under project standards, convert `FC_AlarmManager` to `FB_AlarmManager`.

### OPC-UA on S7-1214 FW4+
- Enable the OPC-UA server at: CPU properties > OPC UA > Server > Activate
- Expose `DB_HIST` and `DB_ALARMS` in the OPC-UA server configuration
- Maximum publishable nodes varies by CPU model — check the CPU manual

### TIA Portal compatibility
This code is written for **TIA Portal V17+** (modern SCL syntax).
It is fully compatible with V20. `RD_SYS_T` requires the standard S7 library (included by default).
