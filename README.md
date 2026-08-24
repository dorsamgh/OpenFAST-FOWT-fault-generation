# OpenFAST Fault Generation for a 15 MW Floating Offshore Wind Turbine

This repository documents how a set of realistic actuator and mooring faults were generated for a 15 MW floating offshore wind turbine using OpenFAST. It accompanies a fault diagnosis dataset built for deep-learning-based condition monitoring research.

## Reference system

- **Turbine:** IEA-15-240-RWT (15 MW reference wind turbine)
- **Platform:** UMaine VolturnUS-S semi-submersible floating platform
- **Simulation tool:** OpenFAST v3.3.0

## Overview

Four distinct fault types were implemented, spanning the turbine's actuation and mooring subsystems. Three of the four required direct modification of the OpenFAST/MoorDyn Fortran source code and recompilation of the solver; the fourth was implemented entirely through ServoDyn's built-in input file configuration, with no source code changes.

| Fault | Full name | Affected subsystem | Implementation | Severity levels |
|---|---|---|---|---|
| **NYL** | Nacelle Yaw Lock | ServoDyn (yaw control) | Source code modification (`UserSubs.f90`) | 0.2°, 0.5°, 1.0° |
| **AD** | Anchor Displacement | MoorDyn (mooring line 1) | Source code modification (`MoorDyn_Line.f90`) | 0.2 m, 0.5 m, 1.0 m |
| **OPB** | Fairlead Out-of-Plane Bending | MoorDyn (mooring line 1, fairlead end) | Source code modification (`MoorDyn_Line.f90`) | 1%, 5%, 10% (relative tension oscillation amplitude) |
| **BPL** | Blade Pitch Lock | ServoDyn (blade 1 pitch control) | Input file configuration only, no source code change | Single configuration (not graded) |

All faults are triggered at simulation time **t = 400 s**, after the turbine has reached steady-state operation, and each is implemented in isolation so its individual signature can be cleanly observed in the resulting dataset.

## What each fault represents

- **NYL** simulates a seized yaw drive/brake. From t = 400 s, the nacelle is commanded to a fixed yaw offset and no longer tracks wind direction changes.
- **AD** simulates a mooring anchor slipping on the seabed. From t = 400 s, the unstretched length of the segment nearest the anchor on line 1 is permanently increased, altering the line's catenary geometry and effective stiffness.
- **OPB** simulates chain-link locking at the fairlead. From t = 400 s, the tension and internal damping force at the fairlead-end segment of line 1 are modulated by a sinusoidal factor centered on the original pretension, reproducing the periodic tension fluctuation characteristic of this phenomenon.
- **BPL** simulates a frozen pitch actuator. From t = 400 s, blade 1 stops tracking pitch control commands and holds at the exact angle it had at that instant, while blades 2 and 3 continue under standard control.

See the README inside each fault's folder for the full technical description, including the exact code changes, the physical reasoning behind them, and the severity level parameterization.

## Repository structure
