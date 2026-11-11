# Drive parameter lists, MONET/N, 2026-11-10

Complete parameter lists (Drive Manager export, UTF-16 CSV: IDN; name; actual; set; min; default; max) of the elevation, azimuth
and derotator drive, exported by Tim at 12:43 on the MONET/N control PC. This is the first export on MONET/N; it is the reference
for the update to BROTLib 0.20.0 / MONETcommon 0.19.2 / HalfBROT 0.14.0 and for the first automatic Wake&Shake test.

State at export: not recorded (activation / power-on state unknown). Fill in.

Startup lists (`* Startup List.csv`, exported from the Startup tab at 12:51 the same day) of the three drives are in this folder.
Their tuning rows (S-0-0100/0101/0104/0106) agree with the drive lists above. The repo-root `Elevation Startup List.csv` /
`Azimuth Startup List.csv` (2024) do not, and are stale.

P-0-0165 (Wake&Shake) was read in the Drive Manager and exported at 12:55 (`* Ch A 0160.csv` in this folder). The values the
drives hold (they agree with the startup lists; this supersedes the earlier word-position decoding). The derotator has no
P-0-0165 entry.

| P-0-0165 | Elevation N | Azimuth N | MONET/S tuned (Nov 2026) |
|---|---|---|---|
| Command mode | Wake and Shake | Wake and Shake | Wake and Shake |
| Activation | **Manual** | **Manual** | On enable request |
| Torque phase 1 / 2 | 50 % / 50 % | 50 % / 50 % | 60 % / 60 % |
| Ramp up time phase 1 / 2 | 500 ms / 500 ms | 1000 ms / 500 ms | not compared |
| Commutation pos control Kp | **1.00** (factory 0.04) | 0.04 (factory) | **0.00** |
| First phase waiting time after step | 150 ms (factory) | 150 ms (factory) | 500 ms |
| P-0-0057 electrical commutation offset | 270.00 deg | 270.00 deg | |

Elevation Kp 1.00 is 25 times the factory value. On MONET/S even 0.04 made the second phase swing 6-8 deg on the elevation
(MotorTuning/WakeAndShake-findings.md). MONET/N has none of the MONET/S tuning for the automatic route.

Differences to the MONET/S export of 2026-11-09 (set values, tuning only; motors differ, do not copy values between telescopes):

| IDN | Elevation N | Elevation S | Azimuth N | Azimuth S |
|---|---|---|---|---|
| S-0-0100 velocity loop P | 12000 | 20000 | 33000 | 120000 |
| S-0-0101 velocity loop integral | 20 ms | 20 ms | 60 ms | 30 ms |
| S-0-0104 position loop Kv | 0.06 | 0.08 | 0.06 | 0.08 |
| S-0-0106 current loop P | 14.4 V/A | 20.2 V/A | 27.3 V/A | 40.5 V/A |

Derotator (N only): S-0-0100 0.097, S-0-0101 8 ms, S-0-0104 1.00, S-0-0106 215.7 V/A (looks like untuned defaults).

Caution: the repo-root `Elevation Startup List.csv` / `Azimuth Startup List.csv` (2024) do NOT match these drive values (for
example azimuth S-0-0100 25000 in the list, 33000 on the drive), and are identical in MONETN and MONETS. Do not use them as a
reference for what the drives hold.

## Elevation P-0-0165 as used for the baseline Wake&Shake (13:3x, 2026-11-10)

Export: `Elevation (AX5125-0000-0214) parameter list Ch A 0160 after-WS-MONETS-values.csv` (taken after the W&S, values are what the user set in the drive; NOT yet in the tsproj startup list).

| Parameter | Elevation before (12:55 export) | Elevation used for W&S |
|---|---|---|
| Torque 1st / 2nd phase | 50 % | **60 %** |
| Ramp-up 1st / 2nd phase | 500 / 500 ms | 500 / 500 ms |
| Commutation pos control Kp | 1.00 | **0.00** |
| Waiting time after step (1st phase) | 150 ms | **500 ms** |
| Other expert values | default | unchanged |
| P-0-0057 (electrical commutation offset) | 270.00 deg | 270.00 deg (unchanged after W&S) |

These are the MONET/S tuned values. Azimuth was left on the old values (50 %, Kp 0.04..1.00 as exported, ramp 500/500, wait 150 ms) and needed 3 attempts for 0166.
To keep them across a project activation they must be added to the Startup List in the tsproj.

## Velocity loop tuning, 2026-11-11 (in the startup lists, activated)

Elevation and azimuth S-0-0100 raised after a step-response and slew test series; S-0-0101, S-0-0104 and S-0-0106 unchanged.
The startup lists hold the value x1000 (`MBoxUserCmdData` of IDN 100 in `MONETN/MONETN.tsproj`):

| Drive | S-0-0100 before | S-0-0100 now | tsproj bytes |
|---|---|---|---|
| Elevation | 12000 | **18000** | `001bb700` -> `80a81201` |
| Azimuth | 33000 | **45000** | `408af701` -> `40a5ae02` |

Test method: the drives run in mode 11 (position control, lag-less, S-0-0032), so the drive closes the position loop; the NC
controller (Kv default, 100 % feedforward) is identical on MONET/N and MONET/S. Each step was set online in the Drive Manager
and tested with the `WakeAndShake` scope (10 ms PLC channels plus the 2 ms drive channels torque feedback and following
distance, ADS port 27910): tracking at hour angle -1 h, 0.01 deg elevation/azimuth offset steps and back, then four
full-speed slews (el 5 deg/s, az 10 deg/s, 3 deg/s^2) between el 30/az 100 and el 75/az 200.

Offset steps (0.01 deg; recordings `MotorTuning/WakeAndShake_3` base ... `_7` final, not in git):

| | El base (12000) | El 15000 | El 18000 | El 18000, Kv 0.04 | Az base (33000) | Az 45000 |
|---|---|---|---|---|---|---|
| Overshoot | 68-77 % | 68-70 % | 64-70 % | 61-66 % | 58-64 % | 52-57 % |
| Damping ratio | 0.08-0.12 | 0.11-0.12 | 0.11-0.14 | 0.13-0.16 | 0.14-0.17 | 0.18-0.20 |
| Within 10 % after | 1.35 s | 1.05 s | 0.93 s | 0.81-0.93 s | 1.2 s | 1.0 s |
| Torque swing | 24-28 % | 29-31 % | 36-38 % | 32-37 % | 19-21 % | 31-32 % |
| Tracking lag | 0.00046 +- 0.00016 | same | same | **0.00081** +- 0.00016 | 0.00075 +- 0.00022 | 0.00075 +- 0.00016 |

Kv 0.04 was reverted (it nearly doubles the tracking lag for little gain).

Slews, same four slews old vs new gains (`WakeAndShake_10` old, `_9` new):

| | Elevation old -> new | Azimuth old -> new |
|---|---|---|
| Max lag accel / cruise / decel | 0.009 / 0.0076 / 0.0086 -> 0.0065 / 0.0055 / 0.0062 deg | 0.0144 / 0.012 / 0.0148 -> 0.0106 / 0.0093 / 0.011 deg |
| Peak torque accel / decel | 63 / 61 % -> 66 / 60 % | 61 / 60 % -> 62 / 60 % |
| Overshoot at arrival | <= 0.002 deg -> <= 0.0022 deg | <= 0.0022 deg -> <= 0.0022 deg |
| Within 0.001 deg after | 1.8-2.5 s -> 1.5-2.0 s | 2.65-2.9 s -> 2.3-2.7 s |
| Torque ripple at cruise (sd) | 3.7 % -> 5.4 % | 3.1 % -> 3.2 % |

Result: 25-35 % less lag on slews, 15-30 % faster settling after offset steps, steadier azimuth tracking. Not fixed: the
50-70 % overshoot on offset steps hardly depends on P or Kv. Likely cause (hypothesis): an offset reaches the drive as a
one-cycle jump of the set position, and the lag-less mode turns it into a velocity feedforward kick. Normal motion is ramped
and does not ring. Proposed fix: ramp the offsets in MONETcommon (also the set-position jump at track start, about 0.008 deg
on azimuth). Cost of the new gains: elevation torque ripple at full speed +50 % (still far below limits).

## tsproj startup list (edited locally, NOT committed, NOT activated)

Superseded: P-0-0165 is on enable for both axes since MONETN `8814191` (azimuth first-phase wait 500 ms).

`MONETN/MONETN.tsproj`, elevation drive (`Box 3`, `Elevation (AX5125-0000-0214)`), the P-0-0165 `MBoxUserCmdData` blob: torque 50 % -> 60 % (both phases), Kp 1.00 -> 0.00, waiting time 150 -> 500 ms. Activation stays Manual, ramps 500/500 ms, everything else unchanged. Byte change: `f401f401f401f40164000500e1009600b80b8403` -> `5802f4015802f40100000500e100f401b80b8403` (layout decoded from the MONETS commit `0fa8790`). The azimuth blob is untouched (still 50 %, Kp 0.04, ramps 1000/500, waiting 150 ms).
Check after the first activation: Drive Manager read-back of P-0-0165 on elevation must show the values above.
