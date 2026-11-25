Line Length Slackening (LLS) Fault — Source Code Modification
Overview
This fault simulates line length slackening (LLS), a permanent geometric anomaly in which mooring line 1 becomes slack. It serves as a controlled proxy for early anchor drag, seabed settlement, or gradual chain elongation. Once triggered, the unstretched length of the anchor-adjacent segment increases permanently, reducing pretension in the affected line and shifting loads asymmetrically to the remaining lines.

The fault is implemented by modifying the internal logic of OpenFAST's MoorDyn module and recompiling the solver, so that the affected line segment's unstretched length undergoes an abrupt step increase once the fault is triggered.

Base source
OpenFAST version: v3.3.0
Module: MoorDyn
File modified: modules/moordyn/src/MoorDyn_Line.f90
Subroutine modified: Line_GetStateDeriv
Line_GetStateDeriv is the core MoorDyn routine that computes, at every time step, the kinematic and force quantities for each line and returns the state derivatives used by the time integrator. Because it is part of MoorDyn's main solution logic rather than a user-defined hook like UserSubs.f90, the modification is inserted directly into this routine.

What was changed
A conditional block was inserted at the point in Line_GetStateDeriv where end-node kinematics have already been set for the current time step, immediately before the segment length values are used in the subsequent kinematic and force calculations:

IF (t .GT. 400.0 .AND. Line%IdNum == 1) THEN
    Line%l(1) = <new_length>_DbKi
END IF
Where Line%l(1) is the unstretched length of the first segment (the segment directly connected to the anchor) of mooring line 1, and <new_length> is the new fixed segment length in meters corresponding to the fault severity (see below). Line%IdNum == 1 restricts the fault to a single mooring line, and t .GT. 400.0 triggers it only after the simulation time exceeds 400 s.

Behavior
Before t = 400 s (t <= 400): Line%l(1) retains its original reference value (
l
0
=
17.0
 m
), and mooring line 1 behaves identically to the healthy (unmodified) model.
From t = 400 s onward (t > 400): Line%l(1) is overwritten at every time step with a fixed, larger value (
l
0
+
Δ
L
), holding the segment length at this new value for the remainder of the simulation. Because Line%l(1) is the reference (unstretched) length used throughout the routine to compute segment strain, tension, and node kinematics, this change propagates through the coupled catenary solution: the line's geometry, pretension, and restoring stiffness are all recalculated around the longer first segment.
The value t = 400 s was chosen as the fault inception time across the entire dataset, consistent with all other fault types, allowing the turbine to reach steady operating conditions before the fault is introduced. In the diagnostic evaluation pipeline, a 60 s post-inception buffer (400–460 s) is discarded so that classifiers rely strictly on the persistent steady-state fault signature rather than the abrupt injection transient.

Reference configuration and severity levels
In the reference mooring configuration of the IEA-15MW turbine on the VolturnUS-S platform, each mooring line has a total unstretched length of 850 m, discretized into 50 equal segments, giving a baseline segment length of 
l
0
=
17.0
 m
 (Line%l(1) = 17.0 under healthy conditions).

Three separate variants of this subroutine were created, differing only in the numeric value hardcoded for Line%l(1), representing subtle, incipient slackening levels (
Δ
L
∈
0.2
,
0.5
,
1.0
 m
):

Severity level	Step length increase (
Δ
L
)	Line%l(1) value in code
Low	0.2 m	17.2_DbKi
Medium	0.5 m	17.5_DbKi
High	1.0 m	18.0_DbKi
Each severity level required its own separately compiled version of OpenFAST, since the new segment length is hardcoded directly into the Fortran source rather than read from an input file at runtime. All three versions are otherwise identical to each other and to the baseline source, differing only in this single numeric literal.

Isolation of the fault
All other mooring lines, and all aerodynamic, hydrodynamic, and control subsystems, were left unmodified, so that the difference between healthy and faulty runs within each stochastic replicate is attributable solely to the line length slackening of line 1.

Files in this folder
File	Description
MoorDyn_Line.f90	Modified MoorDyn source file (representative severity level: 0.2 m)
