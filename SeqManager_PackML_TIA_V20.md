# Sequence Manager PackML — TIA Portal V20
**Arquitectura: FB_SeqRunner + FB_SeqMaster — Basado en OMAC PackML / ISA-88**
Versión 2.0

---

## Tabla de contenidos
1. [Arquitectura general](#1-arquitectura-general)
2. [PackML — Referencia de estados y transiciones](#2-packml--referencia-de-estados-y-transiciones)
3. [UDTs](#3-udts)
4. [DBs globales](#4-dbs-globales)
5. [FB_SeqRunner](#5-fb_seqrunner)
6. [FB_SeqMaster](#6-fb_seqmaster)
7. [Cómo armarlo en TIA Portal](#7-cómo-armarlo-en-tia-portal)
8. [Cómo usarlo — ejemplos](#8-cómo-usarlo--ejemplos)
9. [Escalado y topologías](#9-escalado-y-topologías)
10. [Notas y limitaciones](#10-notas-y-limitaciones)

---

## 1. Arquitectura general

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
     (standalone — sin Master)
```

### Principios de diseño

- **PackML completo** — estados, transiciones y comandos según OMAC PackML v3.0 / ISA-88.
- **UDT_SeqCtrl estructurado** — sub-structs `Config`, `Commands`, `Status`, `Faults`
  alineados con la convención `UDT_xxxCtrl` del proyecto.
- **FltSeverity por dispositivo** — el Runner decide Hold vs Abort según configuración,
  sin cambios a los FBs de dispositivo existentes.
- **Auto-resume opcional para HELD** — configurable por Runner en `Config.AutoResume`.
- **Política de respuesta del Master** — configurable por sub-secuencia en `Config.SubPolicy`.

---

## 2. PackML — Referencia de estados y transiciones

### Estados

| Estado | Valor | Descripción |
|--------|-------|-------------|
| IDLE | 0 | Listo, esperando CmdStart |
| STARTING | 1 | Verificando condiciones previas |
| EXECUTE | 2 | Corriendo pasos normalmente |
| COMPLETING | 3 | Último paso OK, ejecutando cierre controlado |
| COMPLETE | 4 | Secuencia terminada limpiamente |
| HOLDING | 5 | Falla recuperable detectada, deteniendo controladamente |
| HELD | 6 | Detenido en punto seguro, posición conservada |
| RESUMING | 7 | Volviendo a EXECUTE desde HELD |
| PAUSING | 8 | CmdPause recibido, deteniendo controladamente |
| PAUSED | 9 | Detenido en punto seguro por pausa manual |
| RESUMING_PAUSE | 10 | Volviendo a EXECUTE desde PAUSED (mismo estado, flag diferente) |
| STOPPING | 11 | CmdStop recibido, cierre controlado |
| STOPPED | 12 | Detenido limpiamente, puede reiniciarse |
| ABORTING | 13 | Falla crítica, cierre de emergencia |
| ABORTED | 14 | Requiere CmdReset + verificación manual |

### Transiciones válidas

```
IDLE        →  STARTING    : CmdStart + IntlkReady + IntlkEstop
STARTING    →  EXECUTE     : Todas las condiciones previas OK
STARTING    →  ABORTING    : IntlkEstop = FALSE durante STARTING

EXECUTE     →  HOLDING     : FltAny en dispositivo con FltSeverity = 1 (Hold)
EXECUTE     →  ABORTING    : FltAny en dispositivo con FltSeverity = 2 (Abort)
                             OR IntlkEstop = FALSE
EXECUTE     →  PAUSING     : CmdPause
EXECUTE     →  STOPPING    : CmdStop
EXECUTE     →  COMPLETING  : Último paso completado

HOLDING     →  HELD        : Todos los dispositivos en estado seguro
HELD        →  RESUMING    : CmdResume (manual siempre)
                             OR falla resuelta + Config.AutoResume = TRUE
RESUMING    →  EXECUTE     : Condiciones de reanudación OK

PAUSING     →  PAUSED      : Todos los dispositivos en estado seguro
PAUSED      →  RESUMING    : CmdResume
RESUMING    →  EXECUTE     : Condiciones OK

COMPLETING  →  COMPLETE    : Cierre controlado finalizado
COMPLETE    →  IDLE        : CmdReset

STOPPING    →  STOPPED     : Todos los dispositivos confirmados en base
STOPPED     →  IDLE        : CmdReset

ABORTING    →  ABORTED     : Cierre de emergencia completado
ABORTED     →  IDLE        : CmdReset + IntlkEstop = TRUE

ANY         →  ABORTING    : IntlkEstop = FALSE (excepto desde ABORTED)
```

### Comandos PackML

| Comando | Acción |
|---------|--------|
| CmdStart | IDLE → STARTING |
| CmdStop | EXECUTE/PAUSED/HELD → STOPPING |
| CmdPause | EXECUTE → PAUSING |
| CmdResume | HELD/PAUSED → RESUMING |
| CmdAbort | Cualquier estado → ABORTING |
| CmdReset | COMPLETE/STOPPED/ABORTED → IDLE |
| CmdClear | Limpiar fallas sin cambiar estado |

---

## 3. UDTs

### UDT_DeviceCtrl
> Ampliado con `FltSeverity` para política Hold/Abort.

```scl
TYPE "UDT_DeviceCtrl"
    STRUCT
        // ── Config ───────────────────────────────────────────────────────
        Config : STRUCT
            DevType     : USInt;    // 0=Valve 1=Motor 2=Analog 3=Servo
            Enabled     : Bool;     // ENA — FALSE = ignorado por runner
            FltSeverity : USInt;    // 0=Warning 1=Hold 2=Abort
        END_STRUCT;

        // ── Commands (SeqRunner escribe, FB dispositivo lee) ─────────────
        Commands : STRUCT
            CmdAuto     : Bool;     // Valve: Open/Close — Motor: Start/Stop
            CmdSetpoint : Real;     // Analog/Servo: setpoint objetivo
        END_STRUCT;

        // ── Status (FB dispositivo escribe, SeqRunner lee) ───────────────
        Status : STRUCT
            StReady     : Bool;     // Listo para operar
            StDone      : Bool;     // Llegó al estado objetivo
            FltAny      : Bool;     // Falla activa
        END_STRUCT;
    END_STRUCT;
END_TYPE
```

---

### UDT_SeqStep
> Sin cambios respecto a v1.0 — polimórfico por StepType.

```scl
TYPE "UDT_SeqStep"
    STRUCT
        StepType    : USInt;                    // 0=Action 1=Wait 2=Condition 3=SubSeq
        Timeout     : Real;                     // segundos. 0 = sin timeout
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
> Estructura principal del Runner. Sub-structs alineados con convención del proyecto.

```scl
TYPE "UDT_SeqCtrl"
    STRUCT

        // ── Config (se escribe una vez en startup o desde HMI) ────────────
        Config : STRUCT
            AutoResume      : Bool;     // TRUE = auto-resume de HELD si falla se limpia
            AutoResumeTime  : Real;     // segundos máx para auto-resume (0 = sin límite)
            StopTimeout     : Real;     // segundos para confirmar cierre en STOPPING/ABORTING
            HoldTimeout     : Real;     // segundos máx en HOLDING antes de ir a ABORTING
        END_STRUCT;

        // ── Commands (el caller o HMI escribe) ────────────────────────────
        Commands : STRUCT
            CmdStart    : Bool;
            CmdStop     : Bool;
            CmdPause    : Bool;
            CmdResume   : Bool;
            CmdAbort    : Bool;
            CmdReset    : Bool;
            CmdClear    : Bool;
        END_STRUCT;

        // ── Status (el Runner escribe) ────────────────────────────────────
        Status : STRUCT
            State       : USInt;    // Estado PackML actual (ver constantes)
            StStep      : Int;      // Paso actual
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
            StDone      : Bool;     // Secuencia completó todos los pasos limpiamente
        END_STRUCT;

        // ── Faults (el Runner escribe) ────────────────────────────────────
        Faults : STRUCT
            FaultCode   : USInt;    // 0=None 1=StepTimeout 2=EStop 3=DevFault 4=HoldTimeout
            FaultStep   : Int;
            FaultDevice : Int;      // Índice del dispositivo que causó la falla
            FltAny      : Bool;
        END_STRUCT;

        // ── Interlocks (el caller escribe cada ciclo) ─────────────────────
        Interlocks : STRUCT
            IntlkReady  : Bool;     // Permisos de proceso OK
            IntlkEstop  : Bool;     // E-Stop OK (TRUE = sin estop)
        END_STRUCT;

    END_STRUCT;
END_TYPE
```

---

### UDT_MasterSubConfig
> Política de respuesta del Master ante fallas de sub-secuencias.

```scl
TYPE "UDT_MasterSubConfig"
    STRUCT
        OnFaultPolicy   : USInt;    // 0=AbortAll 1=AbortOnly 2=HoldAll 3=HoldOnly
        OnTimeoutPolicy : USInt;    // misma lógica
        IsMandatory     : Bool;     // FALSE = falla de este runner no afecta a los demás
    END_STRUCT;
END_TYPE
```

---

### UDT_MasterCtrl
> Control del Master. Misma convención Config/Commands/Status/Faults.

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
            FaultSubSeq     : USInt;    // Índice de sub-secuencia que causó la falla
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

## 4. DBs globales

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
        // Agregar según proyecto
    END_VAR
END_DATA_BLOCK
```

---

## 5. FB_SeqRunner

**Número sugerido:** FB211

```scl
FUNCTION_BLOCK "FB_SeqRunner"

// ─────────────────────────────────────────────────────────────────────────────
// INTERFAZ
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
    SubHeld     : ARRAY[1..8]  OF Bool;     // Feedback: sub-secuencia en HELD
    SubPaused   : ARRAY[1..8]  OF Bool;     // Feedback: sub-secuencia en PAUSED
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
    _holdFromPause  : Bool;     // Distinguir RESUMING desde HELD vs PAUSED
END_VAR

VAR_TEMP
    _n          : Int;
    _bitVal     : DWORD;
END_VAR

VAR CONSTANT
    // Estados PackML
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
// CUERPO
// ─────────────────────────────────────────────────────────────────────────────

// ── CmdAbort y E-Stop: prioridad máxima, cualquier estado ────────────────────
IF (Ctrl.Commands.CmdAbort AND NOT _prevAbort)
   OR (NOT Ctrl.Interlocks.IntlkEstop AND Ctrl.Status.State <> ST_ABORTED) THEN
    Ctrl.Status.State       := ST_ABORTING;
    Ctrl.Faults.FaultCode   := USInt#2;   // EStop/Abort
    Ctrl.Faults.FltAny      := TRUE;
END_IF;

// ── Máquina de estados PackML ────────────────────────────────────────────────
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
        // Verificar que todos los dispositivos habilitados no tienen fallas
        // y están listos para operar
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
        // ── Detectar fallas en dispositivos ──────────────────────────────
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
            // Hold — falla recuperable
            Ctrl.Status.State       := ST_HOLDING;
            Ctrl.Faults.FaultCode   := USInt#3;
            Ctrl.Faults.FaultDevice := _faultDevice;
            Ctrl.Faults.FaultStep   := Ctrl.Status.StStep;
            Ctrl.Faults.FltAny      := TRUE;
            _holdFromPause          := FALSE;
        ELSIF _faultSeverity = USInt#2 THEN
            // Abort — falla crítica
            Ctrl.Status.State       := ST_ABORTING;
            Ctrl.Faults.FaultCode   := USInt#3;
            Ctrl.Faults.FaultDevice := _faultDevice;
            Ctrl.Faults.FaultStep   := Ctrl.Status.StStep;
            Ctrl.Faults.FltAny      := TRUE;
        END_IF;

        // ── Ejecutar paso (solo si seguimos en EXECUTE) ───────────────────
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

                // StepType 2: Condition externa
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

            // ── Timeout del paso ──────────────────────────────────────────
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

            // ── Avanzar paso ──────────────────────────────────────────────
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

        // Transiciones por comando
        IF Ctrl.Commands.CmdPause AND NOT _prevPause THEN
            Ctrl.Status.State  := ST_PAUSING;
            _holdFromPause     := TRUE;
        END_IF;
        IF Ctrl.Commands.CmdStop AND NOT _prevStop THEN
            Ctrl.Status.State := ST_STOPPING;
        END_IF;

    // ── COMPLETING ───────────────────────────────────────────────────────────
    ST_COMPLETING:
        // Cierre controlado: comandar estado base a todos los dispositivos
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                Devices[_n].Commands.CmdAuto     := FALSE;
                Devices[_n].Commands.CmdSetpoint := 0.0;
            END_IF;
        END_FOR;

        // Verificar que todos confirmaron estado base
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
        // Comandar estado seguro sin perder StStep
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

        // Timeout de HOLDING → ABORTING si no se estabiliza
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

        // Auto-resume: si la falla se limpió y Config.AutoResume = TRUE
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
        // Verificar condiciones antes de retomar
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
        // Cierre de emergencia — más agresivo, sin esperar confirmación de posición
        FOR _n := 1 TO nDevices DO
            IF Devices[_n].Config.Enabled THEN
                Devices[_n].Commands.CmdAuto     := FALSE;
                Devices[_n].Commands.CmdSetpoint := 0.0;
            END_IF;
        END_FOR;

        _tStopTout(
            IN := TRUE,
            PT := DINT_TO_TIME(REAL_TO_DINT(Ctrl.Config.StopTimeout * 1000.0)));

        // En ABORTING no esperamos confirmación de dispositivos — solo timeout
        IF _tStopTout.Q THEN
            Ctrl.Status.State := ST_ABORTED;
            _tStopTout(IN := FALSE, PT := T#0MS);
        END_IF;

    // ── ABORTED ──────────────────────────────────────────────────────────────
    ST_ABORTED:
        // Reset solo si E-Stop OK y comando de reset
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

// ── Memorias de flanco ────────────────────────────────────────────────────────
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

**Número sugerido:** FB210

```scl
FUNCTION_BLOCK "FB_SeqMaster"

VAR_INPUT
    nSubSeqs    : Int;
END_VAR

VAR_IN_OUT
    Ctrl        : "UDT_MasterCtrl";
    SubCtrl     : ARRAY[1..8] OF "UDT_SeqCtrl";    // Ctrl de cada Runner
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
// CUERPO
// ─────────────────────────────────────────────────────────────────────────────

// ── E-Stop y CmdAbort globales ────────────────────────────────────────────────
IF (Ctrl.Commands.CmdAbort AND NOT _prevAbort)
   OR NOT Ctrl.Interlocks.IntlkEstop THEN
    Ctrl.Status.State := ST_ABORTING;
    FOR _n := 1 TO nSubSeqs DO
        SubCtrl[_n].Commands.CmdAbort := TRUE;
    END_FOR;
END_IF;

// ── Detectar fallas en sub-secuencias ─────────────────────────────────────────
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
                1:  // AbortOnly — solo el runner fallido, los demás continúan
                    Ctrl.Faults.FaultSubSeq     := _faultSub;
                    Ctrl.Faults.FltAny          := TRUE;
                2:  // HoldAll
                    Ctrl.Status.State           := ST_HELD;
                    Ctrl.Faults.FaultSubSeq     := _faultSub;
                    Ctrl.Faults.FltAny          := TRUE;
                    FOR _n := 1 TO nSubSeqs DO
                        SubCtrl[_n].Commands.CmdStop := TRUE;
                    END_FOR;
                3:  // HoldOnly — solo el runner fallido
                    Ctrl.Faults.FaultSubSeq     := _faultSub;
                    Ctrl.Faults.FltAny          := TRUE;
            END_CASE;
        END_IF;
    END_IF;
END_FOR;

// ── Máquina de estados del Master ─────────────────────────────────────────────
CASE Ctrl.Status.State OF

    ST_IDLE:
        IF Ctrl.Commands.CmdStart AND NOT _prevStart
           AND Ctrl.Interlocks.IntlkReady
           AND Ctrl.Interlocks.IntlkEstop THEN
            Ctrl.Status.State  := ST_EXECUTE;
            Ctrl.Status.StDone := FALSE;
            // Lanzar sub-secuencias según configuración
            FOR _n := 1 TO nSubSeqs DO
                SubCtrl[_n].Commands.CmdStart    := TRUE;
                SubCtrl[_n].Interlocks.IntlkReady  := TRUE;
                SubCtrl[_n].Interlocks.IntlkEstop  := Ctrl.Interlocks.IntlkEstop;
            END_FOR;
        END_IF;

    ST_EXECUTE:
        // Propagar E-Stop a sub-secuencias
        FOR _n := 1 TO nSubSeqs DO
            SubCtrl[_n].Interlocks.IntlkEstop := Ctrl.Interlocks.IntlkEstop;
        END_FOR;

        // Verificar si todas las sub-secuencias completaron
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

// ── Memorias de flanco ─────────────────────────────────────────────────────────
_prevStart  := Ctrl.Commands.CmdStart;
_prevStop   := Ctrl.Commands.CmdStop;
_prevPause  := Ctrl.Commands.CmdPause;
_prevResume := Ctrl.Commands.CmdResume;
_prevAbort  := Ctrl.Commands.CmdAbort;
_prevReset  := Ctrl.Commands.CmdReset;

END_FUNCTION_BLOCK
```

---

## 7. Cómo armarlo en TIA Portal

### Orden de creación

```
1. UDT_MasterSubConfig
2. UDT_DeviceCtrl
3. UDT_SeqStep
4. UDT_SeqCtrl
5. UDT_MasterCtrl
6. DB_DEVICES          (optimized OFF si usás PUT/GET)
7. DB_SEQUENCES
8. FB_SeqRunner [FB211]
9. FB_SeqMaster [FB210]
```

### Inicialización en OB100

```scl
// Configurar runners
"DB_SEQUENCES".Sub1Ctrl.Config.StopTimeout   := 5.0;
"DB_SEQUENCES".Sub1Ctrl.Config.HoldTimeout   := 30.0;
"DB_SEQUENCES".Sub1Ctrl.Config.AutoResume    := FALSE;
"DB_SEQUENCES".Sub1Ctrl.Config.AutoResumeTime := 0.0;

// Configurar política del master por sub-secuencia
"DB_SEQUENCES".MainCtrl.Config.SubPolicy[1].OnFaultPolicy   := USInt#2;  // HoldAll
"DB_SEQUENCES".MainCtrl.Config.SubPolicy[1].OnTimeoutPolicy := USInt#0;  // AbortAll
"DB_SEQUENCES".MainCtrl.Config.SubPolicy[1].IsMandatory     := TRUE;

// Configurar dispositivos
"DB_DEVICES".Valves[1].Config.DevType     := USInt#0;
"DB_DEVICES".Valves[1].Config.Enabled     := TRUE;
"DB_DEVICES".Valves[1].Config.FltSeverity := USInt#1;  // Hold
```

### Conexión con FBs de dispositivo existentes

```scl
// El FB de dispositivo lee Commands y escribe Status — un único IN_OUT
"FB_Valve".Valve_01(
    CmdAuto  := "DB_DEVICES".Valves[1].Commands.CmdAuto,
    StDone   => "DB_DEVICES".Valves[1].Status.StDone,
    StReady  => "DB_DEVICES".Valves[1].Status.StReady,
    FltAny   => "DB_DEVICES".Valves[1].Status.FltAny,
    Enabled  := "DB_DEVICES".Valves[1].Config.Enabled);
```

---

## 8. Cómo usarlo — ejemplos

### Ejemplo 1: Routing de válvulas — 5 routings exclusivos

```scl
// En FB_FillingMaster, lógica de exclusión de routings:
// Solo se envía CmdStart al runner del routing solicitado.
// Handoff controlado: esperar StStopped del activo antes de arrancar el nuevo.

IF #RoutingRequest <> #RoutingActive THEN
    // Paso 1: detener routing activo
    "DB_SEQUENCES".Sub1Ctrl.Commands.CmdStop := (#RoutingActive = 1);
    "DB_SEQUENCES".Sub2Ctrl.Commands.CmdStop := (#RoutingActive = 2);
    // ... etc

    // Paso 2: cuando el activo confirmó parada, arrancar el nuevo
    IF "DB_SEQUENCES".Sub1Ctrl.Status.StStopped OR
       "DB_SEQUENCES".Sub1Ctrl.Status.StIdle THEN
        #RoutingActive := #RoutingRequest;
        "DB_SEQUENCES".Sub1Ctrl.Commands.CmdStart := (#RoutingRequest = 1);
        "DB_SEQUENCES".Sub2Ctrl.Commands.CmdStart := (#RoutingRequest = 2);
    END_IF;
END_IF;
```

### Ejemplo 2: Verificar estado para lógica de máquina

```scl
// Permiso de operación solo cuando secuencia está en EXECUTE y paso >= 3
IF "DB_SEQUENCES".Sub1Ctrl.Status.StExecute AND
   "DB_SEQUENCES".Sub1Ctrl.Status.StStep >= 3 THEN
    "Outputs".PumpPermit := TRUE;
END_IF;

// Alarma al gestor si el runner está en ABORTED
IF "DB_SEQUENCES".Sub1Ctrl.Status.StAborted THEN
    #AlarmSeqAborted    := TRUE;
    #AlarmFaultCode     := "DB_SEQUENCES".Sub1Ctrl.Faults.FaultCode;
    #AlarmFaultStep     := "DB_SEQUENCES".Sub1Ctrl.Faults.FaultStep;
    #AlarmFaultDevice   := "DB_SEQUENCES".Sub1Ctrl.Faults.FaultDevice;
END_IF;
```

### Ejemplo 3: Auto-resume para falla transitoria de válvula

```scl
// En OB100: configurar runner para auto-resume con 5 segundos de confirmación
"DB_SEQUENCES".Sub1Ctrl.Config.AutoResume     := TRUE;
"DB_SEQUENCES".Sub1Ctrl.Config.AutoResumeTime := 5.0;

// Válvula con FltSeverity=1 (Hold): si pierde fbk y lo recupera en < 5s,
// el runner vuelve a EXECUTE automáticamente desde el mismo paso.
// Si no se recupera, queda en HELD hasta CmdResume manual.
```

---

## 9. Escalado y topologías

### Topología A — Máquina simple (standalone Runner)

```
OB1 → FB_SeqRunner → DB_DEVICES.Valves[1..8]
```
Sin Master. Comandos desde HMI directo a `DB_SEQUENCES.Sub1Ctrl.Commands`.

### Topología B — Routings exclusivos (Master como árbitro)

```
FB_SeqMaster
  ├── FB_SeqRunner [Routing_A]  ← solo uno activo a la vez
  ├── FB_SeqRunner [Routing_B]
  └── FB_SeqRunner [Routing_C]
```
Master no corre pasos — solo gestiona exclusión y handoff entre runners.

### Topología C — Proceso complejo con paralelos

```
FB_SeqMaster "Production"
  ├── FB_SeqRunner [Fill]      → paralelo
  ├── FB_SeqRunner [Heat]      → paralelo
  └── FB_SeqRunner [Agitate]   → paralelo, IsMandatory=FALSE
```
Los tres corren simultáneamente. Agitate puede fallar sin abortar la producción.

### Topología D — Jerárquica multi-área

```
FB_SeqMaster "Plant"
  ├── FB_SeqMaster "Area_Filling"
  │    ├── FB_SeqRunner [...]
  │    └── FB_SeqRunner [...]
  └── FB_SeqMaster "Area_CIP"
       ├── FB_SeqRunner [...]
       └── FB_SeqRunner [...]
```
Interlocks cruzados entre masters de área:
```scl
"DB_SEQ_CIP".MainCtrl.Interlocks.IntlkReady :=
    "DB_SEQ_FILLING".MainCtrl.Status.StIdle;  // CIP solo si Filling está idle
```

---

## 10. Notas y limitaciones

### Comandos en el Master: pulsos vs niveles
Los comandos (`CmdStart`, `CmdStop`, etc.) en `UDT_SeqCtrl.Commands` son Bools de nivel.
El Runner detecta el flanco internamente. El HMI puede escribir un pulso corto o un nivel —
ambos funcionan. Recomendado: pulsos desde HMI para evitar que el comando quede "pegado".

### Propagación de comandos Master → Sub
El Master escribe directamente en `SubCtrl[n].Commands`. Esto significa que el Master
y el operador pueden competir si el HMI también escribe en esos campos. Convención
recomendada: el HMI escribe en `MasterCtrl.Commands`, el Master propaga a los runners.
No permitir escritura directa del HMI a `SubCtrl`.

### CmdClear — no implementado en esta versión
PackML define `CmdClear` para limpiar fallas sin cambiar estado. Implementación sugerida:
en estado HELD o ABORTED, `CmdClear` limpia `Faults.*` sin transición de estado.
Agregar como extensión si el HMI lo requiere.

### REAL_TO_TIME en S7-1200
Usar siempre `DINT_TO_TIME(REAL_TO_DINT(valor * 1000.0))` como se muestra en el código.
`REAL_TO_TIME` directo no está disponible en todos los firmware de S7-1200.

### Límite de 32 dispositivos por Runner
Determinado por el ancho del `DWORD` en ActMask/CondMask.
Para S7-1500: reemplazar `DWORD` por `LWORD` y ajustar los SHR para extender a 64 dispositivos.
