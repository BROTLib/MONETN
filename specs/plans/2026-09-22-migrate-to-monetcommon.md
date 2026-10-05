# Migrate MONETN onto the MONETS/MONETcommon pattern

**Status: planned, not started.** Written up as a spec instead of executed directly, so it can be
reviewed and picked up deliberately (this is safety-relevant PLC code for a live telescope).

## Context

During a pass through MONETcommon's 2026-09-20 code review, several fixes landed in MONETcommon's
`FB_MonetTelescopeControl`, `FB_MonetCoverControl`, `FB_MonetPendantControl`, `FB_MonetCabinetControl`,
and `FB_MonetPowerMonitoring` (see MONETcommon issues #7, #8, #9, #11, #12, #13, #14, #21). It turned out
MONETN doesn't actually use any of those library FBs — `MONETN/MONETN/MONETNRuntime/MONETNRuntime.plcproj`
doesn't even reference the MONETcommon library. MONETN has its own vendored/local copies instead:

- `Components/FB_MonetTelescopeControl.TcPOU`
- `Components/FB_MonetCoverControl.TcPOU`
- `Components/FB_MonetPendantControl.TcPOU`
- `Components/FB_CabinetControl.TcPOU` (note: different type name than MONETcommon's `FB_MonetCabinetControl`)
- `POUs/FB_SafetyHandling.TcPOU` (vs MONETcommon's `FB_MonetSafetyHandling`)
- `POUs/FB_PowerMonitoring.TcPOU` (vs MONETcommon's `FB_MonetPowerMonitoring`)

None of the fixes above are live on MONETN today. MONETS, by contrast, already references the
MONETcommon library directly and has a materially more complete `MAIN.TcPOU` (MQTT watchdog,
diagnostics telemetry, a `TelescopeAuxiliary` instance for mirror-temperature sensors that MONETN
lacks entirely).

**Goal**: bring MONETN onto the same code pattern as MONETS — reference MONETcommon's library
instead of vendored forks, and align `MAIN.TcPOU`'s control-flow body with MONETS's — while
preserving every piece of MONETN-specific hardware wiring and site configuration exactly as it is.

**Why this is safe to do as a type-swap for most of it**: every `AT%I*`/`AT%Q*` hardware I/O
declared *inside* the shared FBs (`FB_MonetCoverControl`, `FB_MonetPendantControl`,
`FB_MonetSafetyHandling`/`FB_SafetyHandling`, `FB_MonetCabinetControl`/`FB_CabinetControl`,
`FB_MonetPowerMonitoring`/`FB_PowerMonitoring`) was confirmed identical in name, type, order, and
comment between MONETN's vendored copies and MONETcommon's versions. The one place real
MONETN-specific data is trapped inside a vendored file is the pointing-model coefficients (see
step 2), and the one genuinely new piece of hardware to wire up is `TelescopeAuxiliary` (step 5).

## Steps

### 1. Add the MONETcommon library reference to MONETN's `.plcproj`

In `MONETN/MONETN/MONETNRuntime/MONETNRuntime.plcproj`, add a `PlaceholderReference` block matching
MONETS's (`MONETS/MONETS/MONETSRuntime/MONETSRuntime.plcproj`):

```xml
<PlaceholderReference Include="MONETcommon">
  <DefaultResolution>MONETcommon, * (IAG)</DefaultResolution>
  <Namespace>MONETcommon</Namespace>
</PlaceholderReference>
```

Then resolve it in the TwinCAT Library Manager (XAE) and check for unresolved-reference errors.
MONETS's `.plcproj` also references `Tc3_JsonXml`, which MONETN's doesn't — confirm on load whether
that's actually required (likely pulled in transitively by MONETcommon or `FB_Comm_MQTT_Influx`) or
can be skipped.

### 2. Extract MONETN's pointing-model coefficients into `MAIN.TcPOU`

MONETN's live pointing coefficients currently live hardcoded inside
`Components/FB_MonetTelescopeControl.TcPOU`, lines 32-34:

```
fbPointing			: FB_PointingModelForward(AOFF := 0.0, BNP := 0.41799581014823517, AN_A := -0.0005399238415356825, AE_A := 0.0010943700155200701, NPAE := 0.3540915955287274, 
                                                  EOFF := 0.0, AN_E := -0.0015387935505830619, AE_E := 0.0028384915610841777, TF := 0.07539398326327973);
fbPointingInverse	: FB_PointingModelInversion(fbPointing := fbPointing);
```

(There's also a commented-out, dead alternate coefficient block at lines 53-56 of that file —
ignore it.)

MONETcommon's `FB_MonetTelescopeControl` takes the pointing model as a *parameter*
(`fbPointing : REFERENCE TO FB_PointingModelForward`, `fbPointingInverse : REFERENCE TO
FB_PointingModelInversion`) rather than owning it internally — matching MONETS's `MAIN.TcPOU`
pattern (`VAR CONSTANT` block, lines 61-63 of `MONETS/MONETS/MONETSRuntime/POUs/MAIN.TcPOU`).

Add to MONETN's `MAIN.TcPOU`, in the `VAR CONSTANT` block (after `telescopeConfig`), using
**MONETN's own coefficients above, not MONETS's**:

```
fbPointing			: FB_PointingModelForward(AOFF := 0.0, BNP := 0.41799581014823517, AN_A := -0.0005399238415356825, AE_A := 0.0010943700155200701, NPAE := 0.3540915955287274, 
                                                  EOFF := 0.0, AN_E := -0.0015387935505830619, AE_E := 0.0028384915610841777, TF := 0.07539398326327973);
fbPointingInverse	: FB_PointingModelInversion(fbPointing := fbPointing);
```

### 3. Swap FB types in `MAIN.TcPOU`'s declarations

Keep the same instance names and `comm := fbComm` init, change only the type:

| Instance | From | To |
|---|---|---|
| `SafetyHandling` | `FB_SafetyHandling` | `FB_MonetSafetyHandling` |
| `CabinetControl` | `FB_CabinetControl` | `FB_MonetCabinetControl` |
| `PowerMonitoring` | `FB_PowerMonitoring` | `FB_MonetPowerMonitoring` |
| `CoverControl` | `FB_MonetCoverControl` (local) | `FB_MonetCoverControl` (MONETcommon — automatic once step 6 removes the vendored file) |
| `TelescopeControl` | `FB_MonetTelescopeControl` (local) | `FB_MonetTelescopeControl` (MONETcommon, same mechanism) |
| `PendantControl` | `FB_MonetPendantControl` (local) | `FB_MonetPendantControl` (MONETcommon, same mechanism) |

Add a new instance (absent in MONETN today), matching MONETS line 29:
```
TelescopeAuxiliary  : FB_TelescopeAuxiliary(comm := fbComm);
```
(see step 5 for the FB itself — it needs a MONETN-local file, not a MONETcommon one.)

Add the pointing-model params to the `TelescopeControl(...)` call (matching MONETS's call,
`MAIN.TcPOU` lines 128-129):
```
fbPointing      := fbPointing,
fbPointingInverse := fbPointingInverse,
```
(keep the existing `bEstopTriggered`/`bMainReady` params already added for MONETcommon#11).

### 4. Align `MAIN.TcPOU`'s body with MONETS's pattern

Port these from `MONETS/MONETS/MONETSRuntime/POUs/MAIN.TcPOU`, keeping MONETN's own site-specific
literals (MQTT host `169.254.146.10`, topics `MONETN/...`, `nMinPanLevel := 3000`, calibration
positions, axis limits, roof `min_position_1`/`min_position_2` params that MONETN has and MONETS's
call omits — keep MONETN's as-is, don't drop them to match MONETS):

1. **`TelescopeAuxiliary()` call**, right after the `TelescopeControl` block (MONETS line 134):
   ```
   TelescopeAuxiliary();
   ```
   and in the telemetry block (MONETS line 253):
   ```
   TelescopeAuxiliary.SendTelemetry();
   ```

2. **MQTT watchdog block** (MONETS lines 34-35 declarations, 186-196 body) — copy verbatim, it has
   no site-specific literals itself (it calls `TelescopeControl.bPark`, `RoofControl.Close()`,
   `fbComm.Publish(...)`, all already-declared instances):
   ```
   mqttWatchdog 		: TON := (PT := T#30S);
   bMqttEverConnected	: BOOL := FALSE;
   ```
   ```
   bMqttEverConnected := bMqttEverConnected OR fbComm.bConnected;
   mqttWatchdog(IN := bMqttEverConnected AND NOT fbComm.bConnected);
   IF mqttWatchdog.Q THEN
   	TelescopeControl.bPark := TRUE;
   	RoofControl.Close();
   	fbComm.Publish('electronics', 'base', 'MQTTWatchdog', 'TRIGGERED');
   END_IF
   ```

3. **Diagnostics telemetry block** (MONETS lines 37-53 declarations, 256-327 body) — copy verbatim,
   same reasoning, no site-specific literals:
   ```
   diagnosticsHeartbeat	: TON := (PT := T#30S);
   bDiagHeartbeatTick		: BOOL;
   bLastInterrupted		: BOOL;
   bLastElevationError	: BOOL;
   bLastAzimuthError		: BOOL;
   bLastDerotatorError		: BOOL;
   bLastFocusError			: BOOL;
   bLastElevationEnabled	: BOOL;
   bLastAzimuthEnabled	: BOOL;
   bLastDerotatorEnabled	: BOOL;
   bLastSafetyError		: BOOL;
   bLastSafetyEstop		: BOOL;
   bLastSafetyState		: BOOL;
   bLastTelescopeReady		: BOOL;
   bLastTelescopeBusy		: BOOL;
   bLastTelescopeStopped	: BOOL;
   ```
   (full body: see `MONETS/MONETS/MONETSRuntime/POUs/MAIN.TcPOU` lines 256-327, copy as-is).

4. **`FocusControl(...)` call** — MONETN's currently omits `ptBrakeDelay`/`ptFocusDelay` that
   MONETS passes (`T#30S`/`T#100MS`); `FB_FocusControl`'s own defaults are `T#4S`/`T#500MS`
   (`HalfBROT/HalfBROT/HalfBROT/POUs/FB_FocusControl.TcPOU` lines 14-15) if omitted, which is a
   valid, safe choice, not a compile requirement. **Decide with the site owner** whether MONETN's
   focus/brake timing should also be tuned to MONETS's values or keep the library defaults — don't
   copy MONETS's values silently, this is a physical timing tune, not a pure code-pattern change.

### 5. Wire `TelescopeAuxiliary`'s sensors, confirmed-only

`FB_TelescopeAuxiliary` maps its mirror-temperature sensors as `AT%I*` *inside* the FB itself
(`tempSensorM1`, `tempSensorM1cell`, `tempSensorM2`, `tempSensorFlange` — see
`MONETS/MONETS/MONETSRuntime/POUs/FB_TelescopeAuxiliary.TcPOU`), so — like the other
single-instance-per-PLC hardware FBs in this codebase — it **cannot** simply be referenced from a
shared library; it needs its own MONETN-local file with MONETN's own I/O mapping.

Confirmed via MONETN's `.tsproj` (`MONETN/MONETN/MONETN.tsproj`, lines 3843-3844, 5571, 5621): there
are dangling/unresolved (`RestoreInfo="ANotFound"`) links for
`MAIN.TelescopeControl.TelescopeAuxiliary.tempSensorM1` and `...tempSensorM2`, each backed by a real
EtherCAT PDO entry (`#x6010`/`#x6000`) still present in the hardware config. **This means the M1/M2
sensor hardware physically exists and was previously wired**, but the software symbol it pointed to
(`TelescopeAuxiliary` nested inside the old `TelescopeControl`) no longer exists. No corresponding
entries were found anywhere in the `.tsproj` for `tempSensorM1cell` or `tempSensorFlange` — those two
are likely not physically wired on MONETN.

Action:
1. Create `MONETN/MONETN/MONETNRuntime/Components/FB_TelescopeAuxiliary.TcPOU`, copied from
   MONETS's/HalfBROT's version as a template, but declaring only `tempSensorM1 AT%I* : INT` and
   `tempSensorM2 AT%I* : INT` (drop `tempSensorM1cell`/`tempSensorFlange`, or keep them declared but
   unmapped — TBD by whoever wires this, since dropping changes the FB's public interface vs.
   MONETcommon's/MONETS's sibling and unmapped-but-declared risks an uninitialized read; simplest
   is to keep the declarations but confirm intent before shipping).
2. **Note the path change**: the old dangling links point at
   `MAIN.TelescopeControl.TelescopeAuxiliary.tempSensorM1/M2` (nested under `TelescopeControl`,
   matching the FB's old internal structure). The new instance in step 3 is a MAIN-level sibling,
   `MAIN.TelescopeAuxiliary.tempSensorM1/M2` — a **different symbol path**. TwinCAT will not
   auto-resolve the old dangling links to the new path; whoever does this on site will need to
   manually re-link those two PDO channels to the new symbols in the I/O configuration (Solution
   Explorer → I/O → re-link, or via "Match with Attached IO" if TwinCAT offers it), not just add the
   FB and expect it to reconnect itself.
3. Confirm with someone on site whether `tempSensorM1cell`/`tempSensorFlange` sensors actually exist
   on MONETN's hardware before deciding whether to wire them at all.

### 6. Remove the now-dead vendored files

Once steps 1-5 compile and resolve correctly against the MONETcommon library, delete these files and
remove their corresponding `<Compile Include>` entries from `MONETNRuntime.plcproj`:

- `Components/FB_MonetCoverControl.TcPOU`
- `Components/FB_MonetPendantControl.TcPOU`
- `Components/FB_MonetTelescopeControl.TcPOU` (only after step 2 has extracted its coefficients)
- `Components/FB_CabinetControl.TcPOU`
- `POUs/FB_SafetyHandling.TcPOU`
- `POUs/FB_PowerMonitoring.TcPOU`

Keep `GVLs/Global_Version.TcGVL` and `DUTs/E_ModeLanguage.TcDUT` — genuinely MONETN-specific, no
equivalent elsewhere. Keep the new `Components/FB_TelescopeAuxiliary.TcPOU` from step 5 — it's
MONETN-specific hardware I/O, not something MONETcommon should provide.

### 7. Review behavior changes this migration brings live on MONETN for the first time

These aren't new work — they're MONETcommon fixes from this session finally taking effect on
MONETN. Worth a deliberate read-through before deploying, not a surprise at runtime:

- `phaseOK` now checks all guard signals (not just 4 warnings) and gates `bMainReady` →
  `_PowerOn` (MONETcommon#11).
- Cover move-timeout now actually fires and stops driving a jammed cover, instead of holding it
  driven indefinitely (MONETcommon#9). 60s placeholder value, untested on real hardware.
- `_ParkTelescope`'s emergency-park branch (close brake, disable axes, close covers) is now
  reachable on error — it was dead code before (MONETcommon#8).
- Elevation move velocity defaults to `5.0` deg/s everywhere a move is commanded (Goto/Home/
  Park/PowerOn/Slew) — was `10.0` in MONETN's vendored copy for everything except Slew. Confirmed
  correct on MONETS; **not yet confirmed on MONETN** (MONETcommon#12).
- Zenith guard added to `_GotoTelescope`: a Goto above 89.5° elevation now errors out instead of
  producing a huge azimuth from `COS(90°) ≈ 0` (MONETcommon#13).
- `fbFocus` null-guards removed from `FB_MonetTelescopeControl` — not a behavior change for MONETN,
  since it always wires a real `FocusControl` already (MONETcommon#14).

### 8. Build and on-site verification

- Full TwinCAT build with zero unresolved references before touching hardware.
- On site: verify E-stop and power-monitoring behavior (tracked as MONETcommon#35 — that work
  already covers MONETN's `SafetyHandling.estop`/`ready` wiring, done independently of this
  migration), verify the new cover-timeout doesn't nuisance-trip on MONETN's actual cover drive
  timing, verify `5.0` deg/s elevation move speed is acceptable on MONETN (only confirmed on
  MONETS so far), re-link the `TelescopeAuxiliary` M1/M2 sensor PDOs per step 5, confirm whether
  M1cell/Flange sensors exist before wiring them.
- Recommend testing outside normal observing hours given the amount of behavior now reaching
  MONETN for the first time.

## Verification

No unit tests exist for this library (MONETcommon#20). Verification is: TwinCAT compiles cleanly
with all vendored files removed and the MONETcommon reference resolved; a manual diff confirms
every MONETN-specific numeric/config value (telescope config, pointing coefficients, MQTT
host/topics, drive limits, roof min/max positions) was preserved; the on-site functional checks in
step 8 pass before this is considered done.
