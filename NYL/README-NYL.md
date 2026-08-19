# Nacelle Yaw Lock (NYL) Fault — Source Code Modification

## Overview

This fault simulates a **nacelle yaw lock**, a failure mode in which the
yaw system's actuation drive or brake seizes and stops responding to
control commands. Once locked, the nacelle can no longer track changes
in wind direction, producing a persistent, uncorrectable yaw
misalignment between the rotor plane and the incoming wind.

The fault is implemented by modifying the internal logic of OpenFAST's
ServoDyn module and recompiling the solver, so that no external
yaw-control commands reach the actuator after the fault is triggered.

## Base source

- **OpenFAST version:** v3.3.0
- **Module:** ServoDyn
- **File modified:** `modules/servodyn/src/UserSubs.f90`
- **Subroutine modified:** `UserYawCont`

`UserSubs.f90` is a user-hook source file provided by OpenFAST for
implementing custom control logic. It contains several dummy
placeholder subroutines — one for each user-definable subsystem
(pitch, generator torque, HSS brake, yaw). By default, `UserYawCont`
is an inert placeholder: it always returns zero yaw position and rate
commands (`YawPosCom = 0`, `YawRateCom = 0`), meaning it exerts no
control action and is effectively unused unless explicitly activated.

## What was changed

The body of `UserYawCont` was modified to introduce a **time-triggered,
constant yaw-position offset**, replacing the routine's default
no-op behavior. The modified logic is:

```fortran
YawPosCom  = 0.0
YawRateCom = 0.0

IF (ZTime >= 400.0_DbKi) THEN
    YawPosCom  = <offset>_ReKi * D2R
    YawRateCom = 0.0_ReKi
END IF
```

Where `<offset>` is the fault severity in degrees (see below), and
`D2R` is OpenFAST's built-in degrees-to-radians conversion constant.

### Behavior

- **Before t = 400 s (`ZTime < 400`):** the subroutine behaves exactly
  as the original placeholder — it commands zero yaw position and
  zero yaw rate, so it has no effect on the simulation and the
  turbine operates under its normal (healthy) yaw control.
- **From t = 400 s onward (`ZTime >= 400`):** the subroutine begins
  commanding a **fixed** yaw position (`<offset> * D2R`, converted from
  degrees to radians) with a yaw rate of exactly zero. Because the
  commanded position no longer changes over time, and the commanded
  rate is held at zero, the nacelle is effectively frozen at this
  fixed angular offset for the remainder of the simulation —
  regardless of any subsequent change in wind direction. This
  reproduces the physical signature of a yaw-drive/brake seizure:
  the nacelle no longer tracks the wind, and any further wind
  direction change directly translates into a growing, uncorrected
  yaw misalignment.

The value `t = 400 s` was chosen as the fault inception time across
the entire dataset (consistent with all other fault types), allowing
the turbine to first reach steady operating conditions before the
fault is introduced.

## Severity levels

Three separate variants of this subroutine were created, differing
only in the numeric value of `<offset>` hardcoded in the `IF` block:

| Severity level | `<offset>` value in code | Physical meaning |
|---|---|---|
| Low   | `0.2` | Nacelle locks at a 0.2° fixed offset |
| Medium | `0.5` | Nacelle locks at a 0.5° fixed offset |
| High  | `1.0` | Nacelle locks at a 1.0° fixed offset |

Each severity level required its own separately compiled version of
OpenFAST, since the offset value is hardcoded directly into the
Fortran source rather than read from an input file at runtime. All
three versions are otherwise identical to each other and to the
baseline placeholder, differing only in this single numeric literal.

## Activation in ServoDyn

Modifying the source code alone does not activate this fault — by
default, ServoDyn does not call any user-defined routine for yaw
control. The ServoDyn primary input file includes a yaw-control mode
selector (`YCMode`) that determines which control source governs the
nacelle yaw behavior (e.g., none, built-in simple control, or a
user-defined routine). For this fault to take effect, this selector
must be set to the user-defined-routine option, which routes control
authority for the yaw actuator to `UserYawCont` instead of ServoDyn's
built-in yaw controller. With this setting active, the modified
subroutine above governs yaw behavior for the entire simulation, and
the fault activates automatically once `ZTime` reaches 400 s.

## Files in this folder

| File | Description |
|---|---|
| `UserSubs.f90` | Modified ServoDyn source file (representative severity level: 0.2°) |
