# Leuze DMU IO-Link ultrasonic sensors — TwinCAT 3 driver (Structured Text)

Driver-level reference for reading the Leuze **DMU** ultrasonic distance sensors over **IO-Link** in
TwinCAT 3 (Structured Text): cyclic distance + switch-state read, and acyclic teach / parameter /
diagnostic access. Master-agnostic — any EtherCAT IO-Link master that exposes the device's process
data and gives ADS/AoE access to its service data.

All values below come from the vendor files in [`docs/`](.): the datasheets
(`DS_DMU218-1500-LA-M12_…`, `DS_DMU230-3000-LA-M12_…`), the IODDs (`SW_DMU2xx_309x_IODD1_1/`), and
the Leuze TwinCAT function-block libraries + manuals (`SW_Fkt-Bau_Beckhoff_DMU2xx_…`).

---

## 1. Sensors covered

| | **DMU218-1500/LA-M12** | **DMU230-3000/LA-M12** |
|---|---|---|
| Part no. | 50149545 | 50149549 |
| **IO-Link Device ID** | **3091** | **3093** |
| Vendor ID | 338 (Leuze electronic) | 338 (Leuze electronic) |
| Series | 200 | 200 |
| Measuring range | **150 … 1500 mm** | **300 … 3000 mm** |
| Resolution | 1.0 mm | 1.0 mm |
| Repeatability | 0.07 % (of end value) | 0.07 % (of end value) |
| Linearity error | 0.5 % | 0.2 % |
| Temperature drift | 0.2 %/K | 0.2 %/K |
| Switching hysteresis (default) | 2 mm | 15 mm |
| Ultrasonic frequency | 200 kHz | 125 kHz |
| Opening angle | 16° | 14° |
| Response time | 100 ms | 170 ms |
| Switching frequency | 5 Hz | 3 Hz |
| Readiness delay (power-up) | 300 ms | 300 ms |
| Digital switching outputs | 1 (Q1) | 1 (Q1) — PD also carries Q2 |
| Analog output | 1 × 4…20 mA | 1 × 4…20 mA |
| Supply | 18 … 30 V DC | 18 … 30 V DC |
| Open-circuit current | ≤ 40 mA | ≤ 45 mA |
| Thread / size | M18×1, Ø18 × 55 mm | M30×1.5, Ø30 × 60 mm |
| Connector | M12, 4-pin, A-coded | M12, 4-pin, A-coded |
| Degree of protection | IP 67 | IP 67 |
| Ambient temp (operation) | −25 … 70 °C | −25 … 70 °C |
| MTTF | 643 years | 579 years |

**M12 4-pin /LA pinout (both):** `1 = V+`, `2 = OUT mA (4…20 mA analog)`, `3 = GND`,
`4 = C/Q (IO-Link) / OUT 1`. In IO-Link mode the distance is read on pin 4; the analog output on
pin 2 is independent and not needed by this driver.

**LED indication (both):** LED1 green steady = ready · green flashing = IO-Link communication ·
yellow steady = switching output active.

---

## 2. IO-Link interface

| | DMU218 (3091) | DMU230 (3093) |
|---|---|---|
| Specification | IO-Link **V1.1** | IO-Link **V1.1** |
| COM mode | **COM2** (38.4 kBaud) | **COM2** |
| Min. cycle time | 2.3 ms | 2.3 ms |
| Frame type | 2.2 | 2.2 |
| Profile | Common Profile | Common Profile |
| SIO (standard-IO) fallback | **No** | **No** |
| Process data in (PDin) | 16 bit | 16 bit |
| Process data out (PDout) | — | — |

> **No SIO fallback:** these devices only deliver data while an IO-Link master holds the port in
> COM2 — there is no switching-output behaviour in standard-IO mode. The port must be configured as
> IO-Link.

Engineering once per port: import the IODD, set the port to **IO-Link / COM2**, confirm the scanned
**Device ID = 3091 / 3093**, enable the master's **Data Storage** (so a replaced sensor is
re-parameterised automatically), and link the 16-bit PDin word to a PLC input. For acyclic access
(§6) note your master's **AoE/ADS NetId**.

---

## 3. Cyclic process data (PDin) — bit layout

Both devices send one **16-bit** PDin word. The measured value is in **millimetres** (IODD
`unitCode 1013`); the low bit(s) carry the binary switch state(s). IO-Link transmits MSB-first.

**DMU218 (3091)** — `ProcessDataIn`, 16 bit:

| Bits | Field | Type | Notes |
|---|---|---|---|
| 15 … 4 | Measured value | UInt, 12-bit | mm, value range 0 … 1800 |
| 3 … 1 | reserved | — | |
| 0 | Switch state BDC1 / Q1 | Bool | |

**DMU230 (3093)** — `ProcessDataIn`, 16 bit:

| Bits | Field | Type | Notes |
|---|---|---|---|
| 15 … 2 | Measured value | UInt, 14-bit | mm, value range 0 … 16383 |
| 1 | Switch state BDC2 / Q2 | Bool | reported in PD even though one physical output |
| 0 | Switch state BDC1 / Q1 | Bool | |

The raw measured-value field **is already millimetres** (1 mm/LSB) — no scaling needed, just the
shift/mask to drop the switch bit(s).

---

## 4. Reading the distance in ST

### 4a. Recommended — Leuze process-data parser function
The Leuze libraries (`SW_DMU218_3091.library` / `SW_DMU230_3093.library`, install via *Library
Repository → Install*, then *Add library*) provide a parser **function** that does the bit
extraction for the exact Device ID:

```
F_Leuze_PD_DMU218_3091( aProcessData : ARRAY[..] OF BYTE; nPDMode : INT ) : ST_Leuze_PD_DMU218_3091
F_Leuze_PD_DMU230_3093( aProcessData : ARRAY[..] OF BYTE; nPDMode : INT ) : ST_Leuze_PD_DMU230_3093
```

Return struct:

| Member | Type | Meaning |
|---|---|---|
| `nMeasuredValue` | `UINT` | distance in **mm** |
| `bSwitchStateBdc1Q1` | `BOOL` | switch output 1 |
| `bSwitchStateBdc2Q1` | `BOOL` | switch output 2 *(DMU230 only)* |
| `bError` | `BOOL` | parse / PD invalid |

```iecst
VAR
    aPD_218 : ARRAY[0..1] OF BYTE;   // linked to the port's 16-bit PDin word
    stPD    : ST_Leuze_PD_DMU218_3091;
    nDist   : UINT;                  // mm
END_VAR

stPD  := F_Leuze_PD_DMU218_3091(aProcessData := aPD_218, nPDMode := 0);
nDist := stPD.nMeasuredValue;
```

### 4b. Alternative — manual bit extraction (no library)
If you read the PDin word directly as a `WORD` (host byte order), the value/switch split is:

```iecst
// DMU218 (3091): 12-bit value in bits 15..4, switch in bit 0
nDist_mm    := SHR(wPD, 4) AND 16#0FFF;     // UINT, millimetres
bSwitch1    := wPD.0;

// DMU230 (3093): 14-bit value in bits 15..2, switches in bits 1..0
nDist_mm    := SHR(wPD, 2) AND 16#3FFF;     // UINT, millimetres
bSwitch1    := wPD.0;
bSwitch2    := wPD.1;
```

> If your master maps PDin as a 2-byte array instead of a word, swap bytes first (IO-Link is
> MSB-first: `byte[0]` is the high byte). The library parser in §4a handles this for you — prefer it.

---

## 5. Driver wrapper (one FB per sensor)

A thin function block keeps the rest of the program clean and adds the validity gating a raw parser
doesn't. Suggested interface (implementation is a few lines around the §4a call):

```iecst
FUNCTION_BLOCK FB_DMU_Sensor          // one instance per physical sensor
VAR_INPUT
    aProcessData : ARRAY[0..1] OF BYTE;   // linked PDin word
    bPortValid   : BOOL;                  // master port/WcState OK (TRUE = good)
END_VAR
VAR_OUTPUT
    rDistance_mm : LREAL;   // measured distance, mm
    bSwitch1     : BOOL;
    bSwitch2     : BOOL;    // DMU230 only
    bValid       : BOOL;    // reading is trustworthy this cycle
END_VAR
```

**Validity rules `bValid` should enforce:**
- `bPortValid` TRUE (master reports the port/communication healthy), **and**
- the **300 ms readiness delay** has elapsed since power-up / since comms came back, **and**
- the parser `bError` is FALSE, **and**
- the value is inside the sensor's measuring window (218: 150…1500 mm, 230: 300…3000 mm) — readings
  pinned at the limits usually mean "no echo / no target".

Hold the last good value (or flag stale) when `bValid` is FALSE so downstream logic can decide.
Instantiate one `FB_DMU_Sensor` per sensor and call it each cycle — no globals required; the linked
PDin word and the port-state bit are the only I/O.

---

## 6. Acyclic service data — teach, parameters, diagnostics

For configuration/teach/diagnostics (not needed for the cyclic distance read) the libraries provide
a service **function block** that wraps the master's ADS IO-Link read/write:

```
FB_Leuze_IOL_DMU218_3091   /   FB_Leuze_IOL_DMU230_3093
```

| I/O | Name | Type | Meaning |
|---|---|---|---|
| IN | `bExecute` | `BOOL` | rising edge starts one transfer |
| IN | `bRW` | `BOOL` | FALSE = read, TRUE = write |
| IN | `nPort` | `T_AmsPort` | ADS port of the IO-Link device |
| IN | `sNetId` | `T_AmsNetID` | the IO-Link master's AoE/ADS NetId |
| IN | `nIdxGroup` | `UDINT` | master's IO-Link parameter index group |
| IN | `tTimeOut` | `TIME` | transfer timeout |
| IN/OUT | `stDeviceData` | `ST_Leuze_IOL_DMU2xx_309x` | all device parameters + command fields |
| OUT | `bDone / bBusy / bError` | `BOOL` | transfer status |
| OUT | `stErrorCode` | `ST_Leuze_IOL_Error` | structured error (see §8) |

> ⚠ **One device's service data per master at a time.** If you instance the service FB for several
> sensors on the same master, interlock them so only one `bExecute` is active at any moment.

**Teach / system commands** — write to *System Command* (ISDU **index 2**, subindex 0):

| Code | Action | Code | Action |
|---|---|---|---|
| 64 | Teach Apply | 130 | Restore factory settings |
| 65 / 66 | Single-value teach setpoint 1 / 2 | 161 / 162 | Set analog output lower / upper limit |
| 67–70 | Two-value teach (TP1/TP2 × SP1/SP2) | 163 | Reset diagnosis information |
| 71–74 | Dynamic teach setpoint 1/2 start / stop | 164 / 165 / 166 | Stop / start / single measurement |
| 79 | Teach cancel | | |

Read *Teach State* (index 59) to confirm a teach result (idle / busy / ok / error).

---

## 7. Multiple sensors close together — avoid crosstalk

Both devices support **Multiplex** and **Synchronous** operation to stop neighbouring ultrasonic
sensors from hearing each other's echoes. This is configured through the **Multi-I/O Pin 4**
parameter (ISDU index 70):

| Value | Pin-4 mode |
|---|---|
| 0 | Push-pull switching output |
| 1 | NPN switching output |
| 2 | PNP switching output |
| 3 | Teach-in / analog-output trigger |
| **4** | **Synchronisation** (sensors fire together) |
| **5** | **Multiplex** (sensors fire in sequence) |

With several DMUs in one area, set them to **Multiplex** (cyclic firing — no mutual interference,
slower aggregate update) or **Synchronous** (simultaneous firing for a common time base). Choose per
mounting geometry; document the chosen mode per sensor.

---

## 8. Error handling (`ST_Leuze_IOL_Error`)

The service FB returns a structured error. Block-level (`nBlockError`):

| Code | Meaning |
|---|---|
| 0x0000 | no error |
| 0x8001 | timeout |
| 0x8002 | no parameter selected |
| 0x8003 | ADS read/write block error (see Beckhoff ADS return codes) |

Device / IO-Link level (with `nIndex` / `nSubIndex` of the offending parameter):

| Code | Meaning |
|---|---|
| 0x8011 | index not available |
| 0x8012 | subindex not available |
| 0x8020 | service temporarily unavailable |
| 0x8023 | access denied (read-only / write-only) |
| 0x8030 | value out of range |
| 0x8031 / 0x8032 | value above / below limit |
| 0x8082 | application not ready |

For cyclic data, treat a non-zero master **WcState** (or the parser `bError`) as "no valid reading
this cycle" and apply the §5 validity rules.

---

## 9. Parameter reference (ISDU index map)

Common IO-Link parameters (subindex 0 unless noted). DMU230 adds the BDC2 entries.

| Index | Parameter | Access | Notes |
|---|---|---|---|
| 2 | System Command | W | teach / commands (§6) |
| 12 | Device Access Locks | R/W | |
| 16–22 | Vendor / product name, serial, HW/FW rev | R | identification |
| 24 | Application-specific tag | R/W | free text |
| 32 | Error count | R | |
| 36 | Device status | R | |
| 59 | Teach state | R | teach result |
| 60 | Setpoints BDC1 | R/W | lower/upper switch points, Q1 |
| 61 | Switchpoint config BDC1 | R/W | logic (NO/NC), mode, hysteresis |
| 62 | Setpoints BDC2 | R/W | *DMU230 only* |
| 63 | Switchpoint config BDC2 | R/W | *DMU230 only* |
| 70 | Multi-I/O Pin 4 | R/W | switching / sync / multiplex (§7) |
| 71 | Multi-I/O Pin 2 | R/W | 0 = off, 1 = 0–20 mA, 2 = 4–20 mA, 3 = 0–10 V |
| 72 | Analog range | R/W | scaling of the 4–20 mA output |
| 84 | Process-data measuring limits | R/W | near/far cut-offs |
| 86 | Temperature | R | device temperature |

**Switchpoint config (index 61/63):** logic `0 = NO / 1 = NC`; mode `0 = off / 1 = single point /
2 = window / 3 = two-point / 128 = reflex`; hysteresis `2 … 20 mm`.

---

## 10. Commissioning checklist

1. Port set to IO-Link/COM2; scanned **Device ID = 3091 / 3093**; Data Storage enabled.
2. PDin word linked to a PLC input; `FB_DMU_Sensor` instanced per sensor and called each cycle.
3. Move a target across the range and confirm `rDistance_mm` tracks at 1 mm
   (218: 150…1500, 230: 300…3000).
4. Unplug a sensor → `bValid` drops (WcState/port invalid).
5. If sensors are mounted close together, set Multiplex/Synchronous (index 70) and re-check for
   crosstalk.
6. (Optional) teach switch points via the service FB (index 2, codes 65/66) and read back the teach
   state (index 59).
