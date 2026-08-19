# Fairlead Out-of-Plane Bending (OPB) Fault — Source Code Modification

## Overview

This fault simulates **out-of-plane bending (OPB)** at the fairlead, a
failure mode in which the chain links of a mooring line lock against
each other as they pass through the fairlead, bending sideways out of
their natural plane instead of freely articulating within it. Under
high pretension, large platform motions, and rough or uneven contact
surfaces, adjacent links can jam rather than roll smoothly over one
another, producing highly localized, time-varying tension fluctuations
at the fairlead connection — a known driver of accelerated fatigue and
chain failure near the fairlead.

Rather than modeling the detailed contact and friction mechanics of
link-on-link locking, OPB is represented here by superimposing a
periodic, sinusoidal fluctuation onto the mooring line's fairlead-end
tension, oscillating around the line's original pretension. This
reproduces the characteristic tension signature of OPB (periodic
over/under-tensioning in phase or near-phase with wave and platform
motion) without altering the line's geometry or introducing any
additional contact/friction modeling.

The fault is implemented by modifying the internal logic of OpenFAST's
MoorDyn module and recompiling the solver, so that the tension and
internal damping force computed at the fairlead-end node of the
affected line are scaled by a time-varying sinusoidal factor once the
fault is triggered.

## Base source

- **OpenFAST version:** v3.3.0
- **Module:** MoorDyn
- **File modified:** `modules/moordyn/src/MoorDyn_Line.f90`
- **Subroutine modified:** `Line_GetStateDeriv`

As with the AD fault, `Line_GetStateDeriv` is the core MoorDyn routine
that computes segment tension, damping force, and other kinematic and
force quantities for each line at every time step. The modification
below is inserted directly into this routine's per-segment force
calculation loop, immediately after the segment tension (`Line%T`) and
internal damping force (`Line%Td`) vectors are computed from the line's
elasticity model.

## What was changed

Inside the loop that computes tension and damping force for each
segment (`DO J = 1, 3`, nested within the segment loop over index `I`),
a conditional block was inserted right after `Line%T(J,I)` and
`Line%Td(J,I)` are assigned:

```fortran
IF (t .GT. 400.0 .AND. Line%IdNum == 1 .AND. I == N) THEN
    Line%T(:,I)  = Line%T(:,I)  * (1.0 + <amplitude> * SIN(0.1 * (t - 400.0)))
    Line%Td(:,I) = Line%Td(:,I) * (1.0 + <amplitude> * SIN(0.1 * (t - 400.0)))
END IF
```

Where:
- `I == N` restricts the modification to the **last segment of the
  line — the segment immediately adjacent to the fairlead node**,
  which is where OPB physically occurs.
- `Line%IdNum == 1` restricts the fault to mooring line 1.
- `t .GT. 400.0` activates the fault only after the simulation time
  exceeds 400 s.
- `<amplitude>` is the relative oscillation amplitude corresponding to
  the fault severity (see below).
- `SIN(0.1 * (t - 400.0))` is a sinusoidal perturbation with angular
  frequency 0.1 rad/s, starting from zero phase at the moment of fault
  inception (t = 400 s).

### Behavior

- **Before t = 400 s (`t <= 400`):** the tension and damping force
  vectors of every segment, including the fairlead-end segment of line
  1, are computed exactly as in the original, unmodified model — no
  effect on the simulation.
- **From t = 400 s onward (`t > 400`), and only for the fairlead-end
  segment of line 1 (`I == N`):** both the tension vector `Line%T(:,I)`
  and the internal damping force vector `Line%Td(:,I)` are multiplied,
  at every time step, by the scalar factor
  `1.0 + <amplitude> * SIN(0.1*(t-400.0))`. Because this factor
  oscillates between `1 - <amplitude>` and `1 + <amplitude>` and is
  centered on 1.0, the **time-averaged tension is unchanged** — the
  line's original pretension is preserved — and only a periodic
  oscillatory component is superimposed on top of it. Because the
  scaling is applied as a scalar multiplier to the existing tension
  and damping vectors (rather than to the line's geometry, segment
  length, or stiffness), the **direction of the force is unaffected** —
  only its magnitude oscillates around the baseline value. This
  isolates the effect of the fault to a pure tension fluctuation at
  the fairlead connection, without perturbing the line's shape, the
  platform's mooring geometry, or any control logic — reproducing the
  characteristic signature of OPB while keeping the fault fully
  isolated and its mechanism transparent.

The value `t = 400 s` was chosen as the fault inception time across
the entire dataset, consistent with all other fault types, allowing
the turbine to first reach steady operating conditions before the
fault is introduced. The oscillation frequency (0.1 rad/s) was kept
fixed across all severity levels; only the amplitude varies.

## Severity levels

Three separate variants of this subroutine were created, differing
only in the numeric value of `<amplitude>` hardcoded in the `IF` block,
each representing a different relative magnitude of tension
fluctuation:

| Severity level | Relative amplitude | `<amplitude>` value in code | Physical meaning |
|---|---|---|---|
| Low    | 1%  | `0.01` | Mild tension oscillation at the fairlead |
| Medium | 5%  | `0.05` | Moderate tension oscillation at the fairlead |
| High   | 10% | `0.10` | Severe tension oscillation at the fairlead |

Each severity level required its own separately compiled version of
OpenFAST, since the amplitude value is hardcoded directly into the
Fortran source rather than read from an input file at runtime. All
three versions are otherwise identical to each other and to the
baseline (unmodified) source, differing only in this single numeric
literal.

## Isolation of the fault

All other mooring lines, and all aerodynamic, hydrodynamic, and control
subsystems, were left unmodified, and no change was made to the line's
geometry, unstretched length, or stiffness properties — so that the
effect of this fault on the simulated dataset can be attributed solely
to the oscillatory tension perturbation at the fairlead end of line 1.

## Files in this folder

| File | Description |
|---|---|
| `MoorDyn_Line.f90` | Modified MoorDyn source file (representative severity level: 1% amplitude) |
