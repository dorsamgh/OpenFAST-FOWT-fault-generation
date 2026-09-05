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

Each fault has its own folder in this repository (NYL, AD, OPB, BPL), containing the modified source or input file along with a dedicated README that gives the full technical description: the exact code change, the physical reasoning behind it, and the severity level parameterization.

## Base source

- **Base OpenFAST version:** v3.3.0. All modified files in this repository are meant to replace the corresponding file in a clean v3.3.0 source tree at its original path (e.g., `modules/servodyn/src/UserSubs.f90`, `modules/moordyn/src/MoorDyn_Line.f90`) before recompiling.
- Each fault folder contains only the representative severity level used as the primary case; the other severity levels differ only in a single hardcoded numeric value, as documented in each folder's README.

## Dataset

The full simulation dataset generated using this code, covering all 11 operating classes (1 healthy and 10 faulted, across the severity levels above) and 12 turbulent wind speed conditions per class, is archived on Zenodo.

**DOI:** [https://doi.org/10.5281/zenodo.22017961](https://doi.org/10.5281/zenodo.22017961)

> **Note:** The dataset is currently under restricted access and will be made openly available upon publication of the associated manuscript. The DOI above is permanent and citable regardless of current access status.

## Associated publication

A manuscript describing the fault diagnosis method developed using this dataset is currently under peer review. This section will be updated with the full citation and link once the paper is published.

## License

This repository is distributed under the [MIT License](LICENSE). Note that `UserSubs.f90` and `MoorDyn_Line.f90` are derivative works of OpenFAST/MoorDyn, originally distributed under the Apache License 2.0. See the license headers within those files for the original copyright notices.

## Citation

If you use this code or the associated dataset, please cite:

- **Manuscript:** Mahdigholi, F. *OpenFAST Simulation-Based Dataset for Multi-Fault Diagnosis of a 15 MW Floating Offshore Wind Turbine under Variable Operating Conditions* Manuscript submitted for publication. Citation details will be added upon publication.
- **Dataset:** Mahdigholi, F. *OpenFAST Simulation-Based Dataset for Multi-Fault Diagnosis of a 15 MW Floating Offshore Wind Turbine under Variable Operating Conditions* [dataset]. Zenodo, v1.0, 2026. https://doi.org/10.5281/zenodo.22017961
