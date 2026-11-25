# Prescribed Periodic Tension Disturbance (PTD) Fault — Source Code Modification

## Overview

This fault implements a **prescribed periodic tension disturbance (PTD)** at the fairlead of mooring line 1. Because lumped-mass mooring solvers such as MoorDyn do not resolve local chain-link contact mechanics or bending stiffness, this scenario is formulated transparently as a controlled periodic tension modulation rather than a direct physical bending model. It serves as a benchmark to evaluate whether diagnostic models can detect subtle, localized oscillatory tension anomalies without a permanent shift in mean load.

The fault is implemented by modifying the internal logic of OpenFAST's MoorDyn module and recompiling the solver, so that the axial tension and internal damping force vectors computed at the fairlead-adjacent segment of line 1 are modulated by a time-varying sinusoidal factor once the fault is triggered.

## Base source

- **OpenFAST version:** v3.3.0
- **Module:** MoorDyn
- **File modified:** `modules/moordyn/src/MoorDyn_Line.f90`
- **Subroutine modified:** `Line_GetStateDeriv`

As with the LLS fault, `Line_GetStateDeriv` is the core MoorDyn routine that computes segment tension, damping force, and other kinematic and force quantities for each line at every time step. The modification below is inserted directly into this routine's per-segment force calculation loop, immediately after the segment tension (`Line%T`) and internal damping force (`Line%Td`) vectors are computed from the line's elasticity model.

## What was changed

Inside the loop that computes tension and damping force for each segment (`DO J = 1, 3`, nested within the segment loop over index `I`), a conditional block was inserted right after `Line%T(J,I)` and `Line%Td(J,I)` are assigned:

```fortran
IF (t .GT. 400.0 .AND. Line%IdNum == 1 .AND. I == N) THEN
    Line%T(:,I)  = Line%T(:,I)  * (1.0 + <amplitude> * SIN(0.1 * (t - 400.0)))
    Line%Td(:,I) = Line%Td(:,I) * (1.0 + <amplitude> * SIN(0.1 * (t - 400.0)))
END IF
```

Where:
- `I == N` restricts the modification to the **last segment of the line — the segment immediately adjacent to the fairlead node**.
- `Line%IdNum == 1` restricts the fault to mooring line 1.
- `t .GT. 400.0` activates the disturbance only after the simulation time exceeds 400 s.
- `<amplitude>` is the relative oscillation amplitude corresponding to the fault severity (see below).
- `SIN(0.1 * (t - 400.0))` is a sinusoidal perturbation with angular frequency $\omega = 0.1\text{ rad/s}$, starting from zero phase at fault inception ($t_f = 400\text{ s}$).

### Behavior

- **Before t = 400 s (`t <= 400`):** The tension and damping force vectors of every segment, including the fairlead-end segment of line 1, are computed identically to the unmodified model.
- **From t = 400 s onward (`t > 400`), and only for the fairlead-end segment of line 1 (`I == N`):** Both the axial tension vector `Line%T(:,I)` and the internal damping force vector `Line%Td(:,I)` are multiplied at every time step by the scalar factor `1.0 + <amplitude> * SIN(0.1 * (t - 400.0))`. Because this factor oscillates symmetrically around `1.0`, the **time-averaged tension is preserved** and only a periodic oscillatory component is superimposed. Applying the scaling as a scalar multiplier preserves the force direction and isolates the disturbance to a pure magnitude oscillation at the fairlead connection without altering line geometry, unstretched length, or stiffness.

The value `t = 400 s` was chosen as the fault inception time across the entire dataset, allowing the turbine to reach steady operating conditions before activation. The angular frequency ($\omega = 0.1\text{ rad/s}$) is fixed across all severity levels; only the modulation amplitude varies.

## Severity levels

Three separate variants of this subroutine were created, differing only in the numeric value of `<amplitude>` hardcoded in the `IF` block:

| Severity level | Relative amplitude ($A$) | `<amplitude>` value in code | Description |
| :--- | :--- | :--- | :--- |
| Low | 1% | `0.01` | Mild periodic tension modulation at the fairlead |
| Moderate | 5% | `0.05` | Moderate periodic tension modulation at the fairlead |
| High | 10% | `0.10` | Severe periodic tension modulation at the fairlead |

Each severity level required its own separately compiled version of OpenFAST, since the amplitude value is hardcoded directly into the Fortran source rather than read from an input file at runtime. All three versions are otherwise identical to each other and to the baseline source, differing only in this single numeric literal.

## Isolation of the fault

All other mooring lines, and all aerodynamic, hydrodynamic, and control subsystems, were left unmodified, and no change was made to the line's geometry, unstretched length, or stiffness properties — so that the difference between healthy and faulty runs within each stochastic replicate is attributable solely to the periodic tension disturbance at the fairlead end of line 1.

## Files in this folder

| File | Description |
|---|---|
| `MoorDyn_Line.f90` | Modified MoorDyn source file (representative severity level: 1% amplitude) |
