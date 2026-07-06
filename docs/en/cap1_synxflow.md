# SynxFlow

## Introduction

Welcome to the SynxFlow HPC Hydraulic Reference Manual. This manual introduces the technical concepts for modelling in SynxFlow for the implementation of the hydrology and debris flow modules in watershed- and gully-scale modelling in a high-performance computing (HPC) environment.

## Model description

SynxFlow (*Synergising High-Performance Hazard Simulation with Data Flow*) is an open-source model capable of dynamically simulating floods, landslides, and debris flows using CUDA-compatible GPUs. It has an intuitive and versatile Python interface, and integrates seamlessly with tools such as NumPy, Pandas, and GDAL for efficient input data processing, simulation execution, and result visualisation. Thanks to the acceleration of modern GPUs, SynxFlow can complete large-scale simulations in a matter of minutes, making it an ideal solution for integration into data science workflows and for optimising and accelerating natural disaster risk assessment, supporting both research and enterprise applications (SynxFlow Repository, 2025).

The authors of SynxFlow have extensive experience in the development and application of hydrodynamic models. The algorithms used in the model are highly efficient and robust, employing Godunov-type shock-capturing methods to solve the shallow water equations (SWE: *Shallow Water Equations*). The model's numerical methods are documented in the articles by Xia et al. (2017, 2023) and Xia & Liang (2018).

### Governing equations (SWE)

Hydrodynamic modelling is implemented under a two-dimensional approach, considering the need for an accurate representation of flow dynamics. Under this approach, the shallow water equations (SWE) (Vreugdenhil, 2013) are solved, which are suited to describing the motion of shallow fluids.

The SWE model solves the conservation equations for volume and momentum, also including temporal and spatial acceleration.

**Volume conservation:**

$$\frac{\partial h}{\partial t} + \frac{\partial (hu)}{\partial x} + \frac{\partial (hv)}{\partial y} = S_v$$

**Momentum conservation in X:**

$$\frac{\partial (hu)}{\partial t} + \frac{\partial}{\partial x}\left(hu^2 + \frac{gh^2}{2}\right) + \frac{\partial (huv)}{\partial y} = -gh\frac{\partial z}{\partial x} - \frac{\tau_x}{\rho}$$

**Momentum conservation in Y:**

$$\frac{\partial (hv)}{\partial t} + \frac{\partial (huv)}{\partial x} + \frac{\partial}{\partial y}\left(hv^2 + \frac{gh^2}{2}\right) = -gh\frac{\partial z}{\partial y} - \frac{\tau_y}{\rho}$$

Where:

| Symbol | Description | Units |
|--------|-------------|-------|
| $\eta = z + h$ | Flow surface elevation | m |
| $t$ | Time | s |
| $h$ | Flow depth | m |
| $u, v$ | Velocity components in X and Y | m/s |
| $S_v$ | Source or sink term (rainfall, infiltration) | m/s |
| $g$ | Gravitational acceleration (9.81) | m/s² |
| $\nu_t$ | Turbulent eddy viscosity | m²/s |
| $\tau$ | Total bed stress | N/m² |
| $\rho$ | Density of the water-solid mixture | kg/m³ |
| $R$ | Hydraulic radius | m |
| $S_f$ | Flow surface slope | — |
| $\theta$ | Inclination angle of the velocity vector | rad |

In SynxFlow (Xia et al., 2017), the SWE are solved with efficient and robust algorithms using Godunov-type shock-capturing methods (Toro, 2001). The **HLLC Riemann solver** (*Harten-Lax-van Leer-Contact*) guarantees the correct capture of hydraulic discontinuities (wave fronts, hydraulic jumps) that occur in extreme events with multiple inflow hydrographs and confluences. Time integration uses an explicit Euler scheme with adaptive time-step control via the CFL condition (*Courant-Friedrichs-Lewy*).

### Manning's roughness coefficient

Flow resistance is parameterised using Manning's roughness coefficient $n$ [m⁻¹/³·s], assigned spatially according to the ESA WorldCover v200 land cover classification (10 m, resampled to 5 m). The values used in the Yaku 2023 event simulation (Rímac River watershed) are:

| ESA Class | Land cover | Manning $n$ [m⁻¹/³·s] |
|-----------|-----------|----------------------|
| 10 | Tree cover (forest) | 0.100 |
| 20 | Shrubland | 0.050 |
| 30 | Grassland | 0.040 |
| 40 | Cropland | 0.030 |
| 50 | Built-up (urban) | 0.020 |
| 60 | Bare/sparse vegetation (bare soil / rock) | 0.030 |
| 70 | Snow and ice | 0.000 |
| 80 | Permanent water bodies | 0.030 |
| 90 | Herbaceous wetland | 0.040 |
| 100 | Moss and lichen | 0.010 |
| — | Default (unassigned classes) | 0.035 |

Values are assigned to the model using `set_grid_parameter`:

```python
case_input.set_landcover(landcover)
case_input.set_grid_parameter(manning={
    'param_value':   [0.10, 0.05, 0.04, 0.03, 0.02, 0.03, 0.0, 0.03, 0.04, 0.0, 0.01],
    'land_value':    [  10,   20,   30,   40,   50,   60,  70,   80,   90,  95, 100],
    'default_value': 0.035})
```

## Methodology

### Modelling approach

SynxFlow implements a two-dimensional modelling approach on a regular structured grid (*raster*), where each DEM cell corresponds to a computational cell. The model solves the SWE in each active cell of the domain at each time step, computing inter-cell fluxes using the HLLC Riemann solver.

The available modules are:

| Module | Python function | Description |
|--------|----------------|-------------|
| Flood (*flood*) | `flood.run()` | SWE + rainfall + GA infiltration |
| Debris flow (*debris*) | `debris.run()` | SWE + Coulomb + erosion *+ GA infiltration*\* |
| Landslide (*landslide*) | `landslide.run()` | Granular mass movement |

\* The GA infiltration module was implemented in the *debris* solver as part of the EWS development.

### Model domain and grid

The simulation domain is defined from a digital elevation model (DEM) in GeoTIFF or ASC format. SynxFlow directly uses the resolution and extent of the DEM as the computational grid.

**Grid considerations:**

- **Resolution**: determines cell size and computational cost. At 5 m resolution, the Rímac watershed (Chosica) generates ~8.2 million active cells.
- **NODATA cells**: cells outside the watershed are automatically excluded from computation.
- **Preprocessed DEM**: pit filling and artefact smoothing are recommended before simulation to avoid numerical instabilities.

```python
from synxflow import IO

DEM = IO.Raster('DEM_5m_filled.tif')
case_input = IO.InputModel(DEM, num_of_sections=1,
                           case_folder='/scratch/.../caso/')
```

### Model parameterisation

The models require the following input data:

| Data | Format | Description |
|------|--------|-------------|
| DEM | GeoTIFF / ASC | Preprocessed digital elevation model |
| Land cover | GeoTIFF | Classification for Manning and infiltration |
| Rainfall mask | GeoTIFF | Spatial extent of rainfall |
| Rainfall time series | CSV | Intensity by zone [m/s] vs time [s] |
| Boundary conditions | Python dict | Inflow and outflow hydrographs |
| Infiltration parameters | numpy array | Green-Ampt per cell |
| Debris parameters | numpy array | Friction and erodible depth |

## Application case: Yaku 2023 event — Rímac River watershed

The debris flow module (*debris*) was applied to the middle-lower Rímac River watershed (Chosica, Lima) to model the Yaku event, which occurred between 9 and 16 March 2023. This event triggered simultaneous debris flows and flooding in multiple tributary gullies (Huascarán, Cusipata, Los Cóndores), causing severe damage to road and residential infrastructure in Chosica.

The simulation was configured with:

- **DEM**: SPOT-6 at 5 m resolution (EPSG:32718), pit-filled prior to simulation.
- **Domain**: ~17.5 million total cells, ~8.2 million active (47%).
- **Rainfall**: WRF 1 km forecast, de-accumulated and reprojected from EPSG:4326.
- **Duration**: 192 hours (9 days), hourly output.
- **Boundary conditions**: upstream hydrograph on the Rímac River + rainfall-driven inflows at tributary gullies.

The simulation was used as a back-analysis for calibration and validation of the EWS, comparing simulated depths against virtual gauge records at tributary gullies and the main channel.

## Parametric sensitivity of the Green-Ampt module

The upper Rímac watershed is predominantly bare soil over rock (ESA class 60, >60% of the area above 2000 m a.s.l.), with high susceptibility to erosion and rapid runoff generation. A One-At-a-Time (OAT) sensitivity analysis of the saturated hydraulic conductivity $K_s$ for class 60 showed that:

- Varying $K_s$ between ×0.05 and ×1.00 (relative to Rawls et al., 1983) produces ~4% variation in $h_{max}$ at the main Rímac channel.
- The hydrometric response is dominated by the upstream boundary condition, not by infiltration in the lower reach.
- For semi-arid Andean watersheds, reduction factors of 0.10–0.30 applied to Rawls $K_s$ values are recommended to represent surface sealing and hydrophobicity in dry soils.

## Glossary of terms

| Term | Description |
|------|-------------|
| **SWE** | *Shallow Water Equations* — 2D shallow water equations |
| **DEM** | *Digital Elevation Model* — digital terrain model |
| **GPU** | *Graphics Processing Unit* — graphics processing unit for parallel computation |
| **HPC** | *High-Performance Computing* — high-performance computing (cluster) |
| **CFL** | Courant-Friedrichs-Lewy stability condition |
| **HLLC** | Harten-Lax-van Leer-Contact Riemann solver |
| **Green-Ampt** | Piston-type wetting front infiltration model |
| **Manning** | Empirical hydraulic roughness coefficient |
| **Solver** | GPU computation module that solves the SWE |
| **Gauge** | Virtual measurement point within the simulation domain |
| **Backup** | File storing the model state at a given instant |
