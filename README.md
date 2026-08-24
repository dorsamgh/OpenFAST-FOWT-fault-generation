
- **Base OpenFAST version:** v3.3.0. All modified files above are meant to replace the corresponding file in a clean v3.3.0 source tree at its original path (e.g., `modules/servodyn/src/UserSubs.f90`, `modules/moordyn/src/MoorDyn_Line.f90`) before recompiling.
- Each fault folder shown above contains only the representative severity level used as the primary case; the other severity levels differ only in a single hardcoded numeric value, as documented in each folder's README.

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

- **Code (this repository):** citation details to be added upon publication
- **Dataset:** Mahdigholi, F. *OpenFAST Simulation-Based Dataset for Multi-Fault Diagnosis of a 15 MW Floating Offshore Wind Turbine under Variable Operating Conditions* [dataset]. Zenodo, v1.0, 2026. https://doi.org/10.5281/zenodo.22017961
