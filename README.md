# MONETN

TwinCAT 3 control application for **MONET/N**, the 1.2 m Alt-Az robotic
telescope of the MONET network (Georg-August-Universität Göttingen), located at
**McDonald Observatory, Texas, USA** (lon −104.0217°, lat 30.6714°,
alt 2000 m).

The application controls the telescope mount (Azimuth / Elevation / Derotator
axes), focus, the three mirror covers, the hydraulic pump/brake system, the
observatory roof, power monitoring, cabinet I/O and the TwinSAFE safety chain,
and publishes MQTT/InfluxDB telemetry. It is built on the **BROTLib** core
library, the **HalfBROT** hardware layer and the **MONET Roof** library
(`FB_RoofControl`); the MONET-specific control logic
(`FB_MonetTelescopeControl`, `FB_SafetyHandling`, `FB_CabinetControl`,
`FB_PowerMonitoring`, `FB_MonetHydraulicsControl`, `FB_MonetPendantControl`,
`FB_MonetFocusControl`, `FB_MonetCoverControl`) is carried as **local vendored
copies** of the MONETcommon library blocks (see
[unification note](#unification-with-monetcommon)).

### Name and heritage ("STELLA1")

The project descends from the **STELLA1** telescope control application
(STELLA is the sister AIP/Göttingen robotic telescope; control electronics and
software are shared) and was later renamed/refactored to MONETN. STELLA1
remnants in the repository: the stale `Mappings.xml` still uses the old PLC
instance name `TIPC^STELLA1Runtime^STELLA1Runtime Instance`, the older
`MONET.tsproj` revision references `..\STELLA1\MONET.tsproj` for the drive
modules, and `FB_MonetTelescopeControl` logs `'STELLA1 startup finished'`.

The legacy README stub (`# STELLA1 / Configuration`: PLC `CX-7A03E9`, AMS
Net-ID `5.122.3.233.1.1`, IP `161.72.132.76`) is **out of date / only partially
verifiable**: the PLC device name and IP do not appear in this repository, and
the checked-in projects use the target Net-ID `5.131.251.159.1.1` (relative
addressing).

---

## Repository layout

```
MONETN/
├── MONETN.sln                  # TwinCAT solution (DriveManager, Scope, MONETN)
├── Azimuth Startup List.csv    # SoE parameter startup list – azimuth AX5125
├── Elevation Startup List.csv  # SoE parameter startup list – elevation AX5125
├── DriveManager/               # TwinCAT Drive Manager 2 project + .tcdmdrv exports
│   ├── DriveManager.tcdmproj
│   ├── Azimuth (AX5125-0000-0214).tcdmdrv
│   ├── Elevation (AX5125-0000-0214).tcdmdrv
│   └── Derotator (AX5206-0000-0214).tcdmdrv
├── Scope/                      # TwinCAT ScopeView projects (axis data pool)
├── MONETN/
│   ├── MONETN.tsproj           # Current system project (TwinCAT 3.1.4024.66)
│   ├── MONET.tsproj            # Older revision (STELLA1 TcPrjPath refs; stale)
│   ├── Mappings.xml            # Stale mapping file (STELLA1Runtime; old names)
│   ├── MONETNRuntime/          # PLC project (current)
│   │   ├── MONETNRuntime.plcproj
│   │   ├── PlcTask.TcTTO       # PLC task (10 ms, priority 20, calls MAIN)
│   │   ├── Components/         # FB_MonetTelescopeControl & co. (vendored copies)
│   │   ├── POUs/               # MAIN, FB_PowerMonitoring, FB_SafetyHandling
│   │   ├── VISUs/              # TwinCAT visualizations (11 screens)
│   │   └── GlobalTextList.TcGTLO
│   ├── MONETNTwinSAFE/         # Safety project (TwinSAFE group on EL6910)
│   └── _Boot/                  # Boot-project target marker only
└── README.md
```

---

## Hardware / EtherCAT topology

Two EtherCAT buses:

**Device 5** (mount drives + roof bus): three servo drives — **Azimuth** and
**Elevation** as `AX5125-0000-0214` (SoE; motors **TMA 0530-070-3VD** resp.
**TMA0360-070-3VC**, Heidenhain `1Vpp-36000S-5V` feedback), **Derotator** as
`AX5206-0000-0214` (motor **AM8532-0D20-0000**, SICK EKM36 absolute encoder).
Each drive carries an **AX5805** TwinSAFE safety card (STO). Term 11 (EK1100)
starts the **roof bus** with the same topology as the MONETRoof application:
EL2008 direction/reset outputs, EL1008 counter/limit/fault inputs, EL4004 speed
setpoints, EL1904/EL2904 TwinSAFE I/O.

**Device 1** (telescope bus, coupler EK1200):

| Terminal | Type | Function |
|---|---|---|
| Safety Logic | EL6910 | TwinSAFE PLC (FSoE master, 7 connections) |
| Safety IN / Safety OUT | EL1904 / EL2904 | TwinSAFE safety I/O |
| Power Analysis | EL3483 | 3-phase power monitoring |
| Focus Encoder / Focus Motor | EL5101-0010 / EL7342 | focus axis: incremental encoder + 2-ch DC-motor terminal (Faulhaber 3557K024CR, gear 43:1, 5 mm spindle) |
| Temperature IN / Mirror Temperature IN | EL3202 / EL3164 | cabinet temperature (RTD) / mirror temperatures |
| Hydraulics OUT / IN / IN2 | EL2008 / EL1008 | oil pump, suction pump, oil status signals |
| Mirror Cover IN / OUT | EL1008 / EL2004 | cover limit switches and open/close commands |
| Brake AzEl | EL2004 | hydraulic brake output |
| Front Panel IN / OUT | EL1008 / EL2004 | buttons, key switch, lamps |
| Pendant Control | EL1008 / EL2008 | hand pendant BCD selector, buttons, lamps |
| Power | EL9410 | E-Bus power supply |

**NC axes** (NC-Task 1 SAF, 2 ms / SVB 10 ms): `Fokus` (Id 1), `Azimuth`
(Id 3), `Elevation` (Id 4), `Derotator` (Id 6).

---

## PLC application architecture

The PLC task (`PlcTask`, 10 ms, priority 20) calls `MAIN`, which instantiates
the MQTT communication function block and all subsystem controllers:

```
MAIN
├── fbComm             : FB_Comm_MQTT_Influx          (MQTT + Influx telemetry, BROTLib)
├── SafetyHandling     : FB_SafetyHandling
├── CabinetControl     : FB_CabinetControl
├── PowerMonitoring    : FB_PowerMonitoring
├── RoofControl        : FB_RoofControl               (MONET Roof library)
├── CoverControl       : FB_MonetCoverControl         (I_MirrorCovers)
├── HydraulicsControl  : FB_HydraulicsControl         (HalfBROT)
├── FocusControl       : FB_FocusControl              (HalfBROT)
├── DerotatorControl   : FB_DerotatorControl          (HalfBROT)
├── ElevationControl   : FB_ElevationControl          (HalfBROT)
├── AzimuthControl     : FB_AzimuthControl            (HalfBROT)
├── TelescopeControl   : FB_MonetTelescopeControl     (telescope state machine)
└── PendantControl     : FB_MonetPendantControl
```

### MAIN

`MAIN` wires the telescope configuration (`ST_TelescopeConfig`:
`name := 'MONET/N'`, `diameter := 1.2`, `mount := 'ALT_AZ'`, manufacturer
`'Halfmann, Haidenhain, Beckhoff'`, park positions `azimuthPark := 190`,
`elevationPark := 60`, `derotatorPark := -116.42`, `focusPark := 53.8`), starts
the subsystem controllers and derives the telescope mode from the cabinet key
switch:

- key local → **manual** mode (`E_TelescopeMode.manual`, hydraulics and brake
  clearing operated locally) or `off`;
- key remote → **automatic** mode (`E_TelescopeMode.automatic`).

It also blinks the error lamp according to error priority (safety > axis >
oil > cover/hydraulics) and publishes `electronics/base/MainReady` /
`MasterError` every 5 s.

### Telescope and axes

- `FB_MonetTelescopeControl` (Components/) implements the full Alt-Az
  telescope lifecycle on top of BROTLib's `FB_AltAzTelescopeControl` (see
  [States and commands](#states-and-commands)); it computes JD/LST
  (`DateTime2JD`, `CT2LST`), applies the pointing model
  (`FB_PointingModelForward` / `FB_PointingModelInversion` with fitted
  constants), the derotator position (`F_DerotatorPosition2`) and per-axis
  tracking velocities, and handles the 360° azimuth/derotator wrap, horizon
  guard and command timeouts.
- The axes are controlled by the HalfBROT blocks `FB_AzimuthControl`,
  `FB_ElevationControl` and `FB_DerotatorControl` (brake interlock, homing,
  limit switches, persistent position; `FB_Axis2`/`FB_BaseAxis` from BROTLib).
- `FB_FocusControl` (HalfBROT) drives the focus Faulhaber motor
  (EL7342 + EL5101) with homing, brake lock and warm-restart-safe position.

### Subsystems

- `FB_RoofControl` (MONET Roof library) — observatory roof, two halves × two
  motors, same logic as the MONETRoof application (`I_Roof`, `E_RoofState`);
  configured in `MAIN` (`min_speed=10000`, `max_speed=30000`,
  `acceleration=150`, `max_position=202`, `max_position_diff=3`,
  `limit_slowdown=5`).
- `FB_MonetCoverControl` — the three mirror covers, sequenced open **1→3→2**,
  close **2→3→1**; limit switches are inverted inputs, a cover errors when
  both open and closed are active.
- `FB_HydraulicsControl` (HalfBROT) — oil pump + suction pump + Az/El brake,
  oil monitoring, watchdogs (pressure 30 s, suction 15 s, main pump 10 s,
  hydraulics 140 s), auto shut-off 300 s after brake closed.
- `FB_PowerMonitoring` — 3-phase power-quality monitoring (EL3483), 19
  `telescope/power/*` telemetry fields.
- `FB_CabinetControl` — front-panel buttons/switches, lamps (via `FB_BLINK`),
  cabinet temperature (warning >50 °C, critical >60 °C).
- `FB_MonetPendantControl` — BCD-selector manual hand pendant.
- `FB_SafetyHandling` — TwinSAFE group startup (2 s delay → ErrAck → Restart
  → per-axis STO resets), monitors group/E-stop/EL1904/EL2904 info data.

---

## States and commands

- **Telescope modes** (`E_TelescopeMode`): `off`, `manual`, `automatic`.
- **TCS commands** (`E_TCSCommand`, BROTLib) with fixed precedence `poweron >
  stop > park > gohome > goto > slew > track > no_command`, executed as
  stage-based state machines with per-command timeouts (2 s stop, 2 min
  park/gohome/goto/slew, 12 h poweron/track) and progress events.
- **Derived states**: `bHomed` (all axes calibrated), `bReady` (homed + covers
  open + axes enabled + brake open + no error), `bTracking` (stable after
  5.5 s), `bIsParked`; `fReadyState` (1 ready / 0 / −1 error), `nMotionState`
  (0 stopped, 1 moving, 8 tracking). Auto-park (12 h) and horizon-guard stop
  with go-home protect the telescope.
- **Axis level** (HalfBROT): enable, home/calibrate, jog, position move, stop,
  tracking; states `Calibrated`, `Ready`, `Busy`, `StandStill`, `InMotion`.
- **Roof**: automatic (command inputs) / manual (buttons) / slow mode; global
  and per-roof `open/close/stop/reset`.

---

## Communication and telemetry

`FB_Comm_MQTT_Influx` (BROTLib) connects to the MQTT broker and publishes
telemetry in Influx line protocol:

| Parameter | Value |
|---|---|
| Broker host | `169.254.146.10` (link-local) |
| Port | `1883`, keep-alive 60 s |
| Subscribe topic (commands) | `MONETN/Telescope/SET` |
| Publish topic (telemetry) | `MONETN/Telemetry` |
| Log topic | `MONETN/Log` |

The command layer parses incoming `command` measurements and routes them to the
interfaces (`Telescope`/`AltAzTelescope`, `Focus`, `Roof` — e.g. `dome_open`,
`dome_close`, `dome_stop` for the roof). Telemetry uses the TSI/TCI-style
`telescope`/`dome` measurement domain (`TELESCOPE.*`, `OBJECT.*`,
`POSITION.*`, `AUXILIARY.COVER.REALPOS`) plus `electronics/base`,
`telescope/power/*`, `hydraulics/base/*` and `MONET.ROOF.*` (roof) domains;
rate 1 s while moving, 5 s idle. Events/logs are published to `MONETN/Log` via
`FB_EventLog`.

## Safety (TwinSAFE)

A TwinSAFE application (`MONETNTwinSAFE`, group `TwinSafeGroup1`) runs on the
**EL6910** safety PLC (FSoE master; messages to the AX5805 STO cards, the
EL1904/EL2904 of both buses, and the roof):

- **E-stop**: `FBEstop1` (safeEstop, E-stop inputs with configurable
  deactivation, 1000 ms delay) → **STO on all three drive axes** via the AX5805
  cards; **EDM**: `FBEdm1..4` (safeEdm, 500 ms) monitor the mirror-cover,
  hydraulics and roof contactors.
- **STO state feedback**: `FBDecouple1/2` + `FBAnd1..3` combine the per-axis
  STO states with the group variables; alias devices expose
  `Azimuth/Elevation/Derotator_STOState` and `*_STOReset`.
- **Group ports**: `Run`, `Restart`, `ErrorAcknowledgement`, `ModuleFault`,
  `FbErr`, `ComErr`, `OutErr`, `OtherErr`, `ComStartup`, `FbDeactive`, `FbRun`,
  `InRun`.
- The PLC side (`FB_SafetyHandling`) performs the startup sequence
  (Run/ErrAck/Restart/STO resets); on E-stop `MAIN` disables all axes.

## Visualization

Eleven TwinCAT visualization screens provide the operator HMI: the main
`Visualization` overview, `Telescope`, `Azimuth`, `Elevation`, `Derotator`,
`Focus`, `Cover`, `Hydraulics`, `Pendant`, `Roof` and `Safety` (E-stop/STO
states, EL1904/EL2904 diagnostics). Text resources live in
`GlobalTextList.TcGTLO`.

## Configuration

- **Drive startup lists** (`Azimuth/Elevation Startup List.csv`): SoE parameter
  startup lists for the AX5125 drives — cycle 2000 µs, operation mode 11
  (position), feedback `1Vpp-36000S-5V`, loop gains, commutation offset, limit
  switches; azimuth motor TMA 0530-070-3VD (44 pole pairs, max 140 rpm, stall
  torque 39.7 Nm), elevation motor TMA0360-070-3VC (33 pole pairs, max 180 rpm,
  stall torque 16.2 Nm). The derotator has no startup CSV (DriveManager only).
- **DriveManager** (`DriveManager.tcdmproj`): the three `.tcdmdrv` exports
  bound to Device 5.
- **MAIN constants**: `telescopeConfig` (see above), roof parameters,
  calibration positions (elevation `fCalibPosition := 25.87`, azimuth
  `211.36`), axis ranges (azimuth ≈ −72…495°, derotator −170…300°), focus
  homing position 92.02, pointing-model coefficients.
- Note: the GVL files are **missing from this checkout** (empty `GVLs/`
  folder); the current project maps I/O symbolically via `AT%I*`/`AT%Q*` and
  the inline mappings in `MONETN.tsproj`.

## Dependencies

- **BROTLib**, **AstroBROT** (BROT) — core library, communication, telescope
  control base classes, tracking/pointing functions, shared DUTs.
- **HalfBROT** (BROT) — axis and hardware function blocks.
- **MONET Roof** (`MONET_Roof`) — `FB_RoofControl`/`I_Roof`.
- **MONETcommon** — not referenced by name; its blocks are vendored locally
  (see below).
- Beckhoff system libraries: `Tc2_MC2`, `Tc2_MC2_Drive`, `Tc2_NC`,
  `Tc3_IotBase`/`Tc3_IotCommunicator` (MQTT), `Tc2_Standard`, `Tc2_System`,
  `Tc2_Utilities`, `Tc3_Module`, plus the TwinCAT visualization libraries.

## Unification with MONETcommon

MONETN carries local (vendored) copies of the MONETcommon function blocks in
`MONETNRuntime/Components/` and `MONETNRuntime/POUs/` instead of referencing
the MONETcommon library — the root cause of code drift between MONETN and
MONETS. The unification plan
([BROTLib/MONET_Unification.md](../BROTLib/MONET_Unification.md)) specifies
which copies to delete and replace with the MONETcommon library reference.

## Building and deployment

The solution is built with TwinCAT 3.1 Build 4024.66 in TwinCAT XAE
(configurations for TwinCAT RT (x64/x86), CE7 (ARMV7) and TwinCAT OS). The PLC
project is `MONETNRuntime` (ADS port 851, symbolic mapping), task `PlcTask`
10 ms/priority 20; NC-Task 1 SAF 2 ms / SVB 10 ms. MQTT requires the Tc3 IoT
license. No boot project data is checked in (`_Boot/` carries only the target
marker); the controller boots from TwinCAT's own boot project on the CX.
