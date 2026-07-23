# Alarm Manager — TIA Portal V20
**Arquitectura: FB_AlarmHandler + FC_AlarmManager**
Versión 1.0

---

## Tabla de contenidos
1. [Arquitectura general](#1-arquitectura-general)
2. [UDTs](#2-udts)
3. [DBs globales](#3-dbs-globales)
4. [FB_AlarmHandler](#4-fb_alarmhandler)
5. [FC_AlarmManager](#5-fc_alarmmanager)
6. [Cómo armarlo en TIA Portal](#6-cómo-armarlo-en-tia-portal)
7. [Cómo usarlo — ejemplos](#7-cómo-usarlo--ejemplos)
8. [Escalado a proyectos grandes](#8-escalado-a-proyectos-grandes)
9. [Notas y limitaciones](#9-notas-y-limitaciones)

---

## 1. Arquitectura general

```
OB1 (cada ciclo)
 └── FC_AlarmManager
      ├── FB_AlarmHandler (instancia Motor_01)  ──► DB_ALARMS.Motors[0]  ──► DB_HIST
      ├── FB_AlarmHandler (instancia Motor_02)  ──► DB_ALARMS.Motors[1]  ──► DB_HIST
      ├── FB_AlarmHandler (instancia Sensor_01) ──► DB_ALARMS.Sensors[0] ──► DB_HIST
      ├── ...
      └── CmdAckAll / CmdClear (procesado al final del FC)
```

**Principios de diseño:**
- **Acceso O(1)** al estado de cada alarma — sin búsquedas lineales.
- **Una instancia de FB por alarma** — cada una tiene su propio estado de flanco.
- **Historial opcional** — se desactiva con `EnableHistory := FALSE` para CPUs sin HMI o recursos limitados.
- **Dos capas separadas**: estado activo (PLC lo usa para lógica) e historial (HMI lo lee por OPC-UA/PUT-GET).

---

## 2. UDTs

Crear en TIA Portal en el orden indicado (dependencias entre UDTs).

### UDT_AlarmState
> Estado activo de una alarma individual. Vive en `DB_ALARMS`.

```scl
TYPE "UDT_AlarmState"
    STRUCT
        AlarmID     : DInt;         // Identificador único de la alarma
        Active      : Bool;         // TRUE = alarma activa en este momento
        Acknowledged: Bool;         // TRUE = operador confirmó la alarma
        TimeRaised  : DTL;          // Timestamp de activación (última)
        TimeCleared : DTL;          // Timestamp de desactivación (última)
        Count       : Int;          // Cantidad de veces que se activó
        Severity    : USInt;        // 0=Info  1=Warning  2=Error  3=Critical
        Category    : USInt;        // Categoría: 1=Motor 2=Sensor 3=Safety etc.
        Device      : USInt;        // ID del dispositivo dentro de la categoría
    END_STRUCT;
END_TYPE
```

### UDT_AlarmHistEntry
> Un registro de evento en el historial. Vive en `DB_HIST`.
> Sin campo `Message` — el HMI mapea los textos usando `AlarmID` o la combinación `(Severity, Category, Device)`.

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
        Valid       : Bool;         // FALSE = entrada vacía (slot libre en el ring buffer)
    END_STRUCT;
END_TYPE
```

### UDT_AlarmHistBuffer
> Ring buffer del historial. Tamaño configurable por proyecto.

```scl
TYPE "UDT_AlarmHistBuffer"
    STRUCT
        Head        : Int;          // Próximo índice de escritura
        Tail        : Int;          // Próximo índice de lectura (consumo externo)
        Count       : Int;          // Entradas válidas actualmente en el buffer
        MaxSize     : Int;          // Tamaño máximo (igual que el array declarado)
        Full        : Bool;         // TRUE = buffer lleno, siguiente escritura sobreescribe
        Entries     : Array[0..99] of "UDT_AlarmHistEntry";  // Ajustar tamaño según proyecto
    END_STRUCT;
END_TYPE
```

> **Nota:** Si necesitás más o menos historial, cambiá el array `Entries` a `Array[0..N-1]`
> y ajustá `MaxSize` al mismo valor N en `DB_HIST`. El código del FB no cambia.

---

## 3. DBs globales

### DB_ALARMS — Estado activo por categorías

```scl
DATA_BLOCK "DB_ALARMS"
    VAR
        Motors   : Array[0..99] of "UDT_AlarmState";   // IDs 0..99
        Sensors  : Array[0..99] of "UDT_AlarmState";   // IDs 100..199 (convención)
        Safety   : Array[0..49]  of "UDT_AlarmState";  // IDs 200..249
        General  : Array[0..49]  of "UDT_AlarmState";  // IDs 250..299
        // Agregar categorías según proyecto
    END_VAR
END_DATA_BLOCK
```

> Cada proyecto define solo las categorías que necesita.
> El índice del array es el índice dentro de la categoría, no el AlarmID absoluto.

### DB_HIST — Historial compartido

```scl
DATA_BLOCK "DB_HIST"
    VAR
        Buf : "UDT_AlarmHistBuffer";
    END_VAR
END_DATA_BLOCK
```

> Inicializar `DB_HIST`.Buf.MaxSize := 100 en el primer scan (OB100 o lógica de init).

---

## 4. FB_AlarmHandler

**Número sugerido:** FB100
**Función:** Gestiona el ciclo de vida completo de una alarma individual.

```scl
FUNCTION_BLOCK "FB_AlarmHandler"

// ---------------------------------------------------------------------------
// INTERFAZ
// ---------------------------------------------------------------------------
VAR_INPUT
    IsActive        : Bool;         // Señal de la alarma (del proceso)
    AlarmID         : DInt;         // ID único de esta alarma
    Category        : USInt;        // Categoría (para historial)
    Severity        : USInt;        // Severidad (para historial)
    Device          : USInt;        // Dispositivo (para historial)
    CmdAck          : Bool;         // ACK individual de esta alarma
    EnableHistory   : Bool;         // FALSE = no escribir al historial (ahorra ciclo)
END_VAR

VAR_IN_OUT
    State           : "UDT_AlarmState";         // Entrada en DB_ALARMS (lectura/escritura directa)
    HistBuf         : "UDT_AlarmHistBuffer";    // Ring buffer compartido en DB_HIST
END_VAR

VAR                                             // Estáticas — persisten entre ciclos
    _PrevActive     : Bool;
    _PrevAck        : Bool;
    _HistIdx        : Int;                      // Índice de esta alarma en el historial (para TimeCleared)
    _InHistory      : Bool;                     // TRUE = hay entrada abierta en el historial
END_VAR

VAR_TEMP
    _sysTime        : DTL;
    _i              : Int;
END_VAR

// ---------------------------------------------------------------------------
// CUERPO
// ---------------------------------------------------------------------------

// ── 1. RISING EDGE: IsActive ────────────────────────────────────────────────
IF IsActive AND NOT _PrevActive THEN

    // Actualizar estado activo (O(1) — el caller ya indexó correctamente)
    State.Active        := TRUE;
    State.Acknowledged  := FALSE;
    State.AlarmID       := AlarmID;
    State.Severity      := Severity;
    State.Category      := Category;
    State.Device        := Device;
    State.Count         := State.Count + 1;

    RD_SYS_T(OUT => _sysTime);
    State.TimeRaised    := _sysTime;
    State.TimeCleared   := DTL#1970-01-01-00:00:00;    // Reset timestamp cleared

    // ── Escribir al historial (si está habilitado) ──
    IF EnableHistory THEN
        // Guardar índice donde vamos a escribir para poder actualizar TimeCleared después
        _HistIdx := HistBuf.Head;
        _InHistory := TRUE;

        // Escribir entrada
        HistBuf.Entries[HistBuf.Head].AlarmID       := AlarmID;
        HistBuf.Entries[HistBuf.Head].Severity      := Severity;
        HistBuf.Entries[HistBuf.Head].Category      := Category;
        HistBuf.Entries[HistBuf.Head].Device        := Device;
        HistBuf.Entries[HistBuf.Head].TimeRaised    := _sysTime;
        HistBuf.Entries[HistBuf.Head].TimeCleared   := DTL#1970-01-01-00:00:00;
        HistBuf.Entries[HistBuf.Head].Acknowledged  := FALSE;
        HistBuf.Entries[HistBuf.Head].Valid         := TRUE;

        // Detectar buffer lleno antes de avanzar Head
        IF HistBuf.Count >= HistBuf.MaxSize THEN
            HistBuf.Full := TRUE;
            // Ring buffer lleno: avanzar Tail (descartamos el evento más antiguo)
            HistBuf.Tail := (HistBuf.Tail + 1) MOD HistBuf.MaxSize;
        ELSE
            HistBuf.Count := HistBuf.Count + 1;
            HistBuf.Full  := FALSE;
        END_IF;

        // Avanzar Head
        HistBuf.Head := (HistBuf.Head + 1) MOD HistBuf.MaxSize;
    END_IF;

END_IF;

// ── 2. FALLING EDGE: IsActive ───────────────────────────────────────────────
IF NOT IsActive AND _PrevActive THEN

    State.Active := FALSE;

    RD_SYS_T(OUT => _sysTime);
    State.TimeCleared := _sysTime;

    // Actualizar TimeCleared en la entrada del historial
    IF EnableHistory AND _InHistory THEN
        HistBuf.Entries[_HistIdx].TimeCleared := _sysTime;
        _InHistory := FALSE;
    END_IF;

END_IF;

// ── 3. ACK individual ───────────────────────────────────────────────────────
IF CmdAck AND NOT _PrevAck THEN
    State.Acknowledged := TRUE;

    // Actualizar Acknowledged en el historial
    IF EnableHistory AND _InHistory THEN
        HistBuf.Entries[_HistIdx].Acknowledged := TRUE;
    END_IF;
END_IF;

// ── 4. Actualizar memorias de flanco ────────────────────────────────────────
_PrevActive := IsActive;
_PrevAck    := CmdAck;

END_FUNCTION_BLOCK
```

---

## 5. FC_AlarmManager

**Número sugerido:** FC100
**Función:** Wrapper que llama todas las instancias de `FB_AlarmHandler` y procesa comandos globales.

```scl
FUNCTION "FC_AlarmManager" : Void

// ---------------------------------------------------------------------------
// INTERFAZ
// ---------------------------------------------------------------------------
VAR_INPUT
    CmdAckAll       : Bool;         // ACK de todas las alarmas activas
    CmdClear        : Bool;         // Limpiar historial completo
    EnableHistory   : Bool;         // Habilitar/deshabilitar historial globalmente
END_VAR

VAR_TEMP
    _i              : Int;
    _sysTime        : DTL;
    _PrevAckAll     : Bool;         // ATENCIÓN: ver nota abajo (*)
    _PrevClear      : Bool;
END_VAR

// ---------------------------------------------------------------------------
// CUERPO
// ---------------------------------------------------------------------------

// ── LLAMADAS A INSTANCIAS DE FB_AlarmHandler ─────────────────────────────
// Una línea por alarma. El índice del array ES el acceso directo O(1).
// Patrón: FB_AlarmHandler.NombreInstancia(
//             IsActive       := <señal del proceso>,
//             AlarmID        := <constante>,
//             Category       := <constante>,
//             Severity       := <constante>,
//             Device         := <constante>,
//             CmdAck         := <bit de ACK individual>,
//             EnableHistory  := EnableHistory,
//             State          := "DB_ALARMS".<Categoria>[<indice>],
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

// ... agregar instancias según el proyecto ...


// ── CmdAckAll — Flanco detectado aquí con VAR estática en el caller ──────
// (*) ATENCIÓN: FC no tiene VAR estáticas. Hay dos opciones:
//
//   OPCIÓN A (recomendada): Mover CmdAckAll y CmdClear a un FB wrapper en lugar de FC.
//   OPCIÓN B: Usar un bit de memoria M o un DB auxiliar para la memoria de flanco.
//
// Ejemplo con Opción B (bit M0.0 = memoria flanco AckAll, M0.1 = memoria flanco Clear):

IF CmdAckAll AND NOT %M0.0 THEN
    FOR _i := 0 TO 99 DO
        "DB_ALARMS".Motors[_i].Acknowledged  := TRUE;
        "DB_ALARMS".Sensors[_i].Acknowledged := TRUE;
    END_FOR;
    FOR _i := 0 TO 49 DO
        "DB_ALARMS".Safety[_i].Acknowledged  := TRUE;
        "DB_ALARMS".General[_i].Acknowledged := TRUE;
    END_FOR;
    // Marcar también el historial
    IF EnableHistory THEN
        FOR _i := 0 TO ("DB_HIST".Buf.MaxSize - 1) DO
            "DB_HIST".Buf.Entries[_i].Acknowledged := TRUE;
        END_FOR;
    END_IF;
END_IF;
%M0.0 := CmdAckAll;

// ── CmdClear — Limpiar historial ─────────────────────────────────────────
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

> **Recomendación:** Si CmdAckAll y CmdClear vienen del HMI con pulsos cortos, convertir
> `FC_AlarmManager` en `FB_AlarmManager` para tener VAR estáticas propias y evitar
> el uso de bits de memoria globales.

---

## 6. Cómo armarlo en TIA Portal

### Orden de creación

```
1. UDT_AlarmState          (PLC types > Add new data type)
2. UDT_AlarmHistEntry
3. UDT_AlarmHistBuffer
4. DB_ALARMS               (Global DB — sin optimización de acceso si usás PUT/GET)
5. DB_HIST                 (Global DB — sin optimización de acceso si usás PUT/GET)
6. FB_AlarmHandler  [FB100]
7. FC_AlarmManager  [FC100]
```

### Configuración de DBs para OPC-UA / PUT-GET

Si el HMI accede al historial por **OPC-UA** (S7-1200 FW4+):
- `DB_HIST`: Optimized block access = **ON** ✅ (OPC-UA lo soporta y es más eficiente)

Si usás **PUT/GET** (HMI Siemens clásico, acceso externo):
- `DB_HIST`: Optimized block access = **OFF** ⚠️ (requerimiento de PUT/GET)
- En propiedades del CPU: activar "Permit access with PUT/GET"

### Instancias múltiples del FB

En TIA V15+, podés declarar las instancias como **Multi-instance** dentro del FC/FB padre,
o como **DBs de instancia individuales**. Para proyectos grandes, multi-instance es más limpio.

### Inicialización (OB100 — Startup)

```scl
// En OB100, inicializar MaxSize del historial
"DB_HIST".Buf.MaxSize := 100;  // Debe coincidir con el tamaño del array Entries
"DB_HIST".Buf.Head    := 0;
"DB_HIST".Buf.Tail    := 0;
"DB_HIST".Buf.Count   := 0;
"DB_HIST".Buf.Full    := FALSE;
```

---

## 7. Cómo usarlo — ejemplos

### Ejemplo 1: Máquina simple sin HMI (S7-1214, standalone)

```scl
// En FC_AlarmManager, todas las instancias con EnableHistory := FALSE
"FB_AlarmHandler".PressureHigh(
    IsActive       := "Process".Pressure > 10.0,
    AlarmID        := DInt#1,
    Category       := USInt#1,
    Severity       := USInt#2,
    Device         := USInt#1,
    CmdAck         := "HMI".AckBtn,
    EnableHistory  := FALSE,          // Sin historial — ahorra ciclo
    State          := "DB_ALARMS".General[0],
    HistBuf        := "DB_HIST".Buf);
```

### Ejemplo 2: Leer estado activo en lógica de máquina

```scl
// El PLC consulta el estado directamente — acceso O(1), sin llamar al FB
IF "DB_ALARMS".Motors[0].Active AND
   "DB_ALARMS".Motors[0].Severity >= USInt#2 THEN
    "Outputs".EmergencyStop := TRUE;
END_IF;

// Verificar si todas las alarmas de motores están acknowledged
IF NOT "DB_ALARMS".Motors[0].Active AND
   NOT "DB_ALARMS".Motors[1].Active THEN
    "Outputs".MotorRunPermit := TRUE;
END_IF;
```

### Ejemplo 3: HMI leyendo historial por OPC-UA

El HMI lee `DB_HIST.Buf.Entries[0..N]` y muestra solo las entradas donde `Valid = TRUE`.
Para el texto del mensaje, el HMI tiene una tabla interna:

```
AlarmID 1001 → "Motor 1 — Temperatura alta"
AlarmID 1002 → "Motor 1 — Sobrecarga"
AlarmID 2001 → "Sensor 1 — Presión alta"
```

O alternativamente usa `(Category, Device, Severity)` para componer el texto dinámicamente.

### Ejemplo 4: Agregar una alarma nueva al sistema

1. Agregar la señal de proceso al DB de proceso.
2. Agregar una línea en `FC_AlarmManager`:

```scl
"FB_AlarmHandler".NombreNuevaAlarma(
    IsActive       := "DB_PROCESS".NuevaSenal,
    AlarmID        := DInt#3001,       // ID único, no repetir
    Category       := USInt#3,         // Safety
    Severity       := USInt#3,         // Critical
    Device         := USInt#5,
    CmdAck         := "DB_CMDS".AckNueva,
    EnableHistory  := EnableHistory,
    State          := "DB_ALARMS".Safety[0],  // índice libre
    HistBuf        := "DB_HIST".Buf);
```

3. Agregar el texto en la tabla del HMI. Listo.

---

## 8. Escalado a proyectos grandes

| Escala | Configuración recomendada |
|--------|--------------------------|
| ~50 alarmas (máquina simple) | 4 categorías × 15 entradas, historial 50 entradas, `EnableHistory := FALSE` si no hay HMI |
| ~200 alarmas (línea de producción) | 6–8 categorías, historial 200–500 entradas, OPC-UA activo |
| 500+ alarmas (planta) | Considerar múltiples `FC_AlarmManager` por zona/área, un `DB_HIST` por área o uno global centralizado |
| 1000+ alarmas | Dividir en FCs por zona. El FB no cambia. Solo agregar instancias y categorías en `DB_ALARMS` |

**Memoria estimada por alarma:**
- `UDT_AlarmState`: ~50 bytes por entrada
- `UDT_AlarmHistEntry`: ~40 bytes por entrada
- 500 alarmas activas + 1000 entradas historial ≈ **65 KB** — dentro de cualquier S7-1500.

---

## 9. Notas y limitaciones

### Limitación: `_HistIdx` en activaciones rápidas
Si una alarma se activa/desactiva varias veces antes de que el HMI lea el historial,
`_HistIdx` apunta siempre a la **última** entrada escrita. Las entradas anteriores de esa
alarma quedan con `TimeCleared = 1970` hasta que sean sobreescritas por el ring buffer.
Para sistemas con alarmas de muy alta frecuencia, considerar no actualizar `TimeCleared`
en el historial y manejarlo solo en `State.TimeCleared`.

### Limitación: FC sin VAR estáticas
Los flancos de `CmdAckAll` y `CmdClear` en el FC usan bits de memoria (`%M`).
Si esto no es aceptable por estándares del proyecto, convertir `FC_AlarmManager` a `FB_AlarmManager`.

### OPC-UA en S7-1214 FW4+
- Activar servidor OPC-UA en: CPU properties > OPC UA > Server > Activate
- Exponer `DB_HIST` y `DB_ALARMS` en la configuración del servidor OPC-UA
- Máximo de nodos publicables varía por modelo de CPU — verificar en Manual de la CPU

### Compatibilidad TIA Portal
Este código está escrito para **TIA Portal V17+** (sintaxis SCL moderna).
En V20 es completamente compatible. `RD_SYS_T` requiere librería estándar S7 (incluida por defecto).
