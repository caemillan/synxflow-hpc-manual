# Introduction

## About this manual

This **Reference Manual for Hydrodynamic Simulation of Natural Hazards on HPC with SynxFlow** documents the technical foundations, computing environment configuration, and usage procedures for the SynxFlow hydrodynamic model for simulating floods and debris flows in Andean watersheds.

This manual has been prepared by **Carlos Millán-Arancibia** (Servicio Nacional de Meteorología e Hidrología del Perú — SENAMHI, Peru) and **Xilin Xia** (University of Birmingham, UK) as part of the development of the Early Warning System (EWS) for the middle-lower Rímac River watershed. It is aimed at professionals in hydrology, hydraulic engineering, and earth sciences who need to implement 2D hydrodynamic simulations with GPU acceleration on high-performance computing environments.

## Scope

The manual covers:

1. **Description of the SynxFlow model** — foundations of the shallow water equations (SWE), available modules, and simulation capabilities.
2. **HPC environment configuration** — installation, compilation, and execution on NVIDIA GPU clusters.
3. **Debris flow module** — governing equations, Coulomb rheological model, erosion/deposition, and coupled infiltration.
4. **Green-Ampt infiltration module** — GPU implementation, parameters by land cover class, and considerations for semi-arid Andean watersheds.

## Context: Early Warning System for the Rímac River

The Rímac River watershed, with its outlet at the city of Lima (population ~10 million), is one of the watersheds with the highest hydrometeorological risk in Peru. The Chosica area concentrates the exposure to debris flows generated in tributary gullies (Huascarán, Cusipata, Los Condores) during intense rainfall events associated with phenomena such as Coastal El Niño.

The Yaku event (9–16 March 2023) triggered debris flows and overflows in multiple gullies, severely affecting road and residential infrastructure in Chosica. The simulations documented in this manual correspond to the modelling of that event as a back-analysis case for calibration and validation of the EWS.

### EWS Components

```
FTP (WRF 1 km rainfall)
       │
       ▼
Rain preprocessing
(deaccumulation, reprojection, clipping to watershed)
       │
       ▼
GPU Simulation — SynxFlow (debris solver + Green-Ampt)
DEM 5m · 8.2 million cells · ~20 min to 7h depending on event
       │
       ▼
Postprocessing
(h_max, v_max, hazard level HR = h·(v+0.5) + 0.5)
       │
       ▼
Web tile publication (FTP → SILVIA cartographic server)
```

### Simulation domain

| Parameter | Value |
|-----------|-------|
| Simulation area | Bajo Rímac — Chosica to Lima |
| DEM | SPOT-6, 5 m resolution, EPSG:32718 |
| Total cells | ~17.5 million |
| Active cells | ~8.2 million (47%) |
| Extent | ~23.9 km (E-W) × 18.3 km (N-S) |

## Manual structure

| Chapter | Contents |
|---------|---------|
| **Ch. 1** | Introduction: EWS context, simulation domain, manual structure |
| **Ch. 2** | Description of SynxFlow: SWE equations, modules, methodology, case studies |
| **Ch. 3** | HPC environment configuration: installation, SLURM, reference runtimes |
| **Ch. 4** | Debris flow module: equations, Coulomb friction, erosion, parameters |
| **Ch. 5** | Green-Ampt infiltration module: GPU implementation, Rawls parameters, Andean watersheds |
| **References** | Full bibliography cited in the manual |
