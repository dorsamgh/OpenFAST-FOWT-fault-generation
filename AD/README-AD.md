# Anchor Displacement (AD) Fault — Source Code Modification

## Overview

This fault simulates an **anchor displacement**, a failure mode in which
one of the mooring line anchors loses its holding capacity and slips or
shifts position on the seabed. Once displaced, the effective geometry of
the affected mooring line is permanently altered: the unstretched length
of the segment resting on the seabed increases, which changes the line's
catenary shape, pretension, and effective stiffness — even though the
anchor itself is not physically simulated as a moving body.

The fault is implemented by modifying the internal logic of OpenFAST's
MoorDyn module and recompiling the solver, so that the affected line
segment's unstretched length is permanently increased once the fault is
triggered.

## Base source

- **OpenFAST version:** v3.3.0
- **Module:** MoorDyn
- **File modified:** `modules/moordyn/src/MoorDyn_Line.f90`
- **Subroutine modified:** `Line_GetStateDeriv`

`Line_GetStateDeriv` is the core MoorDyn routine that computes, at every
time step, the kinematic and force quantities for each line and returns
the state derivatives used by the time integrator. It is not a
user-defined hook like `UserSubs.f90` — it is part of MoorDyn's main
solution logic, so the modification below is inserted directly into
this routine rather than into a placeholder subroutine.

## What was changed

A conditional block was inserted at the point in `Line_GetStateDeriv`
where end-node kinematics have already been set for the current time
step, immediately before the segment length values are used in the
subsequent kinematic and force calculations:

```fortran
IF (t .GT. 400.0 .AND. Line%IdNum == 1) THEN
    Line%l(1) = <new_length>_DbKi
END IF
```

Where `Line%l(1)` is the unstretched length of the first segment
(the segment directly connected to the anchor) of mooring line 1, and
`<new_length>` is the new fixed segment length in meters corresponding
to the fault severity (see below). `Line%IdNum == 1` restricts the
fault to a single mooring line, and `t .GT. 400.0` triggers it only
after the simulation time exceeds 400 s.

### Behavior

- **Before t = 400 s (`t <= 400`):** `Line%l(1)` retains its original,
  reference value, and mooring line 1 behaves exactly as in the healthy
  (unmodified) model — no effect on the simulation.
- **From t = 400 s onward (`t > 400`):** `Line%l(1)` is overwritten
  every time step with a fixed, larger value. Because this assignment
  happens unconditionally at every step once the time threshold is
  passed, the segment length is held at this new value for the
  remainder of the simulation rather than transitioning gradually.
  Since `Line%l(1)` is the reference (unstretched) length used
  throughout the rest of the routine to compute segment strain,
  tension, and node kinematics, this single change propagates through
  the entire catenary solution: the line's geometry, pretension, and
  angle of entry at the fairlead are all recalculated around this new,
  permanently longer first segment. Physically, this reproduces the
  effect of the anchor point having advanced (moved closer to the
  platform along the seabed), enlarging the portion of the line that
  lies flat on the seabed and reducing the effective restoring
  stiffness of the mooring system — the same outcome that a real
  anchor slip would produce, without needing to simulate the anchor
  itself as a free body.

The value `t = 400 s` was chosen as the fault inception time across
the entire dataset, consistent with all other fault types, allowing
the turbine to first reach steady operating conditions before the
fault is introduced.

## Reference configuration and severity levels

In the reference mooring configuration of the IEA-15MW turbine, the
unstretched length of the seabed-resting portion of each line is 850 m,
divided into 50 equal segments, giving a baseline segment length of
17 m (i.e., `Line%l(1) = 17.0` under healthy conditions).

Three separate variants of this subroutine were created, differing
only in the numeric value hardcoded for `Line%l(1)`, each corresponding
to a different magnitude of anchor displacement added on top of this
17 m baseline:

| Severity level | Anchor displacement | `Line%l(1)` value in code |
|---|---|---|
| Low    | 0.2 m | `17.2_DbKi` |
| Medium | 0.5 m | `17.5_DbKi` |
| High   | 1.0 m | `18.0_DbKi` |

Each severity level required its own separately compiled version of
OpenFAST, since the new segment length is hardcoded directly into the
Fortran source rather than read from an input file at runtime. All
three versions are otherwise identical to each other and to the
baseline (unmodified) source, differing only in this single numeric
literal.

## Isolation of the fault

All other mooring lines, and all aerodynamic, hydrodynamic, and control
subsystems, were left unmodified, so that the effect of this fault on
the simulated dataset can be attributed solely to the anchor
displacement of line 1.

## Files in this folder

| File | Description |
|---|---|
| `MoorDyn_Line.f90` | Modified MoorDyn source file (representative severity level: 0.2 m) |
