# Blade Pitch Lock (BPL) Fault — Input File Configuration

## Overview

This fault simulates a **blade pitch lock**, a failure mode in which
blade 1's pitch actuator stops tracking pitch control commands and
freezes at its current angle, while the other two blades continue to
respond normally. Once locked, blade 1 no longer participates in the
rotor's aerodynamic power/load regulation, and the resulting load
asymmetry between blades — coupled with platform motion — is the
signature this fault is intended to reproduce.

Unlike the NYL, AD, and OPB faults, **this fault required no
modification to the OpenFAST source code and no recompilation.** It is
implemented entirely through the standard pitch-maneuver override
capability already built into the ServoDyn input file, by configuring
the parameters that ServoDyn exposes for this exact purpose.

## Base source

- **OpenFAST version:** v3.3.0
- **Module:** ServoDyn (input file only — no source modification)
- **File used:** `IEA-15-240-RWT-UMaineSemi_ServoDyn.dat`
- **Representative wind condition:** 13 m/s turbulent wind

## What was configured

ServoDyn's pitch control section includes a built-in mechanism for
overriding standard pitch control with a scripted pitch maneuver on a
per-blade basis, triggered at a specified simulation time and driven
toward a specified final angle at a specified rate. This mechanism —
normally intended for simulating events such as an emergency feathering
maneuver — was repurposed here to emulate a locked blade by commanding
blade 1 to move, at an extremely high rate, to the exact pitch angle it
already held at the moment of fault inception. The relevant parameters
in the ServoDyn input file are:

| Parameter | Value used | Meaning |
|---|---|---|
| `PCMode` | `5` | Pitch control mode: user-defined from Bladed-style DLL — i.e., the standard external pitch controller governs normal operation, exactly as in the healthy baseline |
| `TPitManS(1)` | `400` | Time (s) at which the override pitch maneuver for blade 1 begins, and at which standard pitch control for blade 1 ends |
| `PitManRat(1)` | `1000.0` | Rate (deg/s) at which the override maneuver drives blade 1's pitch toward its final commanded angle — set extremely high so that the transition is effectively instantaneous |
| `BlPitchF(1)` | `8.989` | Final commanded pitch angle (degrees) for blade 1's override maneuver |
| `TPitManS(2)`, `TPitManS(3)` | `9999.9` | Override maneuver times for blades 2 and 3 — set far beyond the simulation duration so these blades are never affected and continue under standard pitch control throughout |

### Behavior

- **Before t = 400 s:** all three blades, including blade 1, are
  governed entirely by the standard external pitch controller
  (`PCMode = 5`), exactly as in the healthy baseline simulation. No
  override maneuver is active.
- **At t = 400 s (`TPitManS(1)`):** ServoDyn ends standard pitch
  control for blade 1 and begins the override maneuver, driving blade
  1's pitch toward `BlPitchF(1) = 8.989°` at a rate of `1000°/s`.
  Because this rate is far higher than any physically meaningful pitch
  rate, and because `BlPitchF(1)` was set to the exact angle blade 1
  already held at t = 400 s under healthy operation, the maneuver
  completes essentially instantaneously and produces no visible
  transient — blade 1's angle does not jump, it simply stops changing.
- **From t = 400 s onward:** blade 1 remains fixed at `8.989°` for the
  rest of the simulation, regardless of subsequent wind or control
  demands, while blades 2 and 3 continue to be actively pitched by the
  standard controller (`TPitManS(2)` and `TPitManS(3)` never trigger,
  since they are set to `9999.9`, beyond the simulation length). This
  reproduces the physical signature of a locked pitch actuator: one
  blade frozen at a fixed angle while the rotor's other two blades and
  overall control loop continue operating normally, producing the load
  asymmetry characteristic of this fault.

## Determining `BlPitchF(1)`

The value `8.989°` is not an arbitrary or representative constant — it
is the **exact pitch angle blade 1 held at simulation time t = 400 s in
the corresponding healthy-condition simulation**, for the representative
13 m/s turbulent wind case. This value was read directly from the
healthy simulation's blade 1 pitch output at t = 400 s and then
hardcoded into `BlPitchF(1)` for the faulted run, ensuring that the
lock occurs precisely at the angle blade 1 would naturally have been at
when the fault begins, rather than forcing a discontinuous jump to an
unrelated angle.

## Severity levels

This fault has **only a single configuration** — it is not graded into
multiple severity levels like NYL, AD, and OPB. A blade pitch lock is
inherently a binary event (the blade is either tracking control
commands or it is not); there is no natural continuous severity
parameter analogous to a yaw offset, anchor displacement distance, or
tension oscillation amplitude.

## Isolation of the fault

Blades 2 and 3 remain under standard pitch control for the entire
simulation, and no other ServoDyn, MoorDyn, or aerodynamic/hydrodynamic
settings were altered from the healthy baseline configuration — so the
effect of this fault on the simulated dataset can be attributed solely
to blade 1's pitch lock.

## Files in this folder

| File | Description |
|---|---|
| `IEA-15-240-RWT-UMaineSemi_ServoDyn.dat` | ServoDyn input file with the BPL override maneuver configured (representative case: 13 m/s turbulent wind) |
