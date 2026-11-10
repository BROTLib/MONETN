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

## tsproj startup list (edited locally, NOT committed, NOT activated)

`MONETN/MONETN.tsproj`, elevation drive (`Box 3`, `Elevation (AX5125-0000-0214)`), the P-0-0165 `MBoxUserCmdData` blob: torque 50 % -> 60 % (both phases), Kp 1.00 -> 0.00, waiting time 150 -> 500 ms. Activation stays Manual, ramps 500/500 ms, everything else unchanged. Byte change: `f401f401f401f40164000500e1009600b80b8403` -> `5802f4015802f40100000500e100f401b80b8403` (layout decoded from the MONETS commit `0fa8790`). The azimuth blob is untouched (still 50 %, Kp 0.04, ramps 1000/500, waiting 150 ms).
Check after the first activation: Drive Manager read-back of P-0-0165 on elevation must show the values above.
