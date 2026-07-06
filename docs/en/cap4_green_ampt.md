# Green-Ampt Infiltration Module

## Theoretical background

SynxFlow's infiltration module is based on the Green-Ampt (1911) model, an analytical solution derived from Darcy's law for a piston-type wetting front. The model assumes that the soil above the front is completely saturated and the soil below maintains its initial moisture content.

The instantaneous infiltration rate $f(t)$ [m/s] is expressed as:

$$f(t) = K_s \left(1 + \frac{\psi \cdot \Delta\theta}{F(t)}\right)$$

Where:

| Symbol | SynxFlow parameter | Description | Units |
|--------|--------------------|-------------|-------|
| $K_s$ | `hydraulic_conductivity` | Effective saturated hydraulic conductivity | m/s |
| $\psi$ | `capillary_head` | Average capillary suction at the wetting front | m |
| $\Delta\theta = \theta_s - \theta_i$ | `water_content_diff` | Moisture deficit (fillable porosity) | — |
| $F(t)$ | `cumulative_depth` | Cumulative infiltration | m |

As $F(t)$ grows, the infiltration rate decreases asymptotically toward $K_s$. In the long term (saturated soil), all infiltration capacity is given by $K_s$.

## GPU implementation (CUDA)

In SynxFlow, infiltration is calculated at each active cell of the domain and at each time step of the solver. The analytical solution for a time step $\Delta t$ is obtained by implicitly solving the ODE $dF/dt = f(F)$:

$$F_1 = \frac{1}{2}\left[(F_0 + K_s \Delta t) + \sqrt{(F_0 + K_s \Delta t)^2 + 4\,K_s\,\Delta t\,(\psi + h)\,\Delta\theta}\right]$$

Where $F_0$ is the cumulative infiltration at the start of the step, $h$ is the available water depth, and $F_1$ is the cumulative infiltration at the end of the step.

The effective infiltration increment in step $\Delta t$ is limited by the available water depth:

$$\Delta F = \min(h,\ F_1 - F_0)$$

The solver updates each cell:

```
cumulative_depth += ΔF
h               -= ΔF
```

This formulation ensures that a cell does not infiltrate more water than it contains, preventing negative depths.

### CUDA kernel

The GPU kernel implementing this logic operates over all active cells in parallel:

```cpp
// cuda_infiltration.cu — kernel cuInfiltrationGreenAmptKernel
Scalar F_0        = cumulative_depth[index];
Scalar total_head = capillary_head + h;          // psi + h
Scalar disc       = F_0 + K_s * dt;
Scalar F_1        = 0.5 * (disc + sqrt(disc*disc
                    + 4.0*K_s*dt*total_head*delta_theta));
Scalar delta_F    = fmin(h, F_1 - F_0);

cumulative_depth[index] += delta_F;
h[index]                -= delta_F;
```

The kernel is invoked from the debris module after each temporal integration of the SWE.

## Parameters by land cover class (Rawls et al., 1983)

Green-Ampt parameters are assigned spatially according to the ESA WorldCover land cover classification (10 m), resampled to the simulation DEM. Reference values come from Rawls et al. (1983), Table 4.5.2, obtained for soils from temperate zones in the USA.

| ESA class | Land cover | Soil type | $K_s$ [mm/h] | $\psi$ [cm] | $\Delta\theta$ [-] |
|-----------|-----------|-----------|-------------|------------|-------------------|
| 10 | Tree cover | Silt loam | 6.48 | 16.68 | 0.486 |
| 20 | Shrubland | Sandy loam | 10.92 | 11.01 | 0.412 |
| 30 | Grassland | Loam | 3.38 | 8.89 | 0.434 |
| 40 | Cropland | Loam | 3.38 | 8.89 | 0.434 |
| 50 | Built-up | Impervious | 0.036 | 5.00 | 0.050 |
| 60 | Bare/sparse | Loamy sand | 29.97 | 6.13 | 0.401 |
| 80 | Water bodies | Saturated | 0.0 | 0.0 | 0.000 |
| 90 | Wetland | Clay | 1.08 | 31.63 | 0.385 |

> **Note:** Values in model units (m/s and m) are obtained as:
> $K_s\ [\text{m/s}] = K_s\ [\text{mm/h}] \times 2.778 \times 10^{-7}$
> $\psi\ [\text{m}] = \psi\ [\text{cm}] \times 0.01$

## Considerations for semi-arid Andean watersheds

The Rawls et al. (1983) values were obtained from agricultural soils in temperate zones with complete soil profiles. In semi-arid Andean watersheds (e.g., upper Rímac catchment, Chosica), reduction factors are applied for the following physical reasons:

### Surface sealing (*rain crust*)

The kinetic impact of raindrops breaks up soil aggregates, compacts the surface, and blocks macroscopic pores (*structural sealing*). In soils with fine-grained material (silt, volcanic ash, colluvial material), particles migrate inward and seal the surface pores (*depositional sealing*). Both processes reduce surface $K_s$ by one to two orders of magnitude relative to laboratory values (Assouline, 2004).

Rain crust formation has direct consequences for flow generation:

- **Rapid runoff generation**: by suppressing infiltration within the first minutes of the event, the seal partitions rainfall directly to overland flow, reducing the precipitation depth needed to generate runoff (Assouline, 2004).
- **Increased peak discharge**: the crust acts as a near-impermeable surface, raising the runoff coefficient and the probability of initiating hyperconcentrated and debris flows (Thouret et al., 2020, *Earth-Sci. Rev.* 201:103003).
- **Hardened crusts from previous deposits**: in gullies with recurrent events, deposits from earlier flows form cemented crusts that suppress bed infiltration, facilitating avulsion into secondary channels (Thouret et al., 2020).

This process is particularly relevant for class 60 soils (bare soil / rock) at the onset of the Yaku 2023 event, when the soil was in a dry condition prior to the onset of rainfall.

### Hydrophobicity in dry soils

Soils in semi-arid zones exhibit water repellence (hydrophobicity) when dry, reducing effective infiltration during the first hours of rainfall. Repellence decreases as the soil wets (thresholds reported by Doerr et al., 2000). In volcanic environments and Andean gullies with perennial vegetation, hydrophobicity explains the high frequency of lahars and debris flows during low-intensity rainfall at the onset of the wet season, when dry soil repels water and generates immediate surface runoff (Capra et al., 2010, *J. Volcanol. Geotherm. Res.* 189:105–117).

### Thin soils over rock

In the upper Rímac catchment (above 2000 m.a.s.l.), class 60 covers more than 60% of the area and corresponds to thin soils (<20 cm) over granitic or metamorphic rock. Effective infiltration is limited by the storage capacity of the soil profile, not by $K_s$.

### Recommended reduction factors

For the Rímac watershed (Yaku 2023 event), the OAT (*One-At-a-Time*: one parameter is varied at a time while all others are held fixed) sensitivity analysis on $K_s$ for class 60 suggests:

| Multiplier | $K_s$ class 60 [mm/h] | Justification |
|------------|----------------------|---------------|
| ×1.00 (Rawls) | 29.97 | Undisturbed soil, reference |
| ×0.50 | 14.94 | Soil with moderate surface sealing |
| ×0.18 | 5.39 | Rocky Andean soil, severe crust |
| ×0.10 | 2.99 | Extreme surface sealing / outcropping rock |
| ×0.05 | 1.49 | Rock cover >80% |

> The sensitivity of the hydrometric response to the $K_s$ class 60 multiplier is low in the main Rímac channel (~4% variation in $h_{max}$), indicating that the upstream boundary condition dominates over infiltration in the lower reach of the catchment.

## State variable `cumulative_depth`

`cumulative_depth` ($F$) represents the cumulative infiltration since the start of the simulation [m]. It is a **state variable** with physical meaning:

- $F = 0$: completely dry soil (standard initial condition).
- $F > 0$: soil with antecedent moisture. The initial infiltration rate is lower than with $F=0$.
- $F \to \infty$: saturated soil; $f \to K_s$ (steady state).

### Initialisation with antecedent moisture

For events preceded by prior rainfall, the initial condition $F=0$ underestimates antecedent moisture and overestimates infiltration capacity. In the case of Yaku 2023, the soil was in a relatively dry condition before the onset of rainfall, so $F=0$ is a reasonable approximation for that event. This value should be adjusted when records or estimates of significant antecedent rainfall are available:

```python
# Dry soil (default)
case_input.set_grid_parameter(cumulative_depth=0.0)

# Pre-wet soil (equivalent to estimated antecedent F)
F_antecedente = 0.02  # m, equivalent to ~20 mm of prior rainfall
case_input.set_grid_parameter(cumulative_depth=F_antecedente)
```

### `cumulative_depth` outputs

SynxFlow writes `cumulative_depth` as:
- Periodic raster: `cumulative_depth_{t}.asc` (every output interval).
- Backup file: `cumulative_depth_backup_{t}.dat` (every backup interval).

This allows cascaded simulations to be resumed and the spatial distribution of cumulative infiltration at the end of the event to be analysed.

## Parameter assignment in Python

```python
# Green-Ampt parameters by ESA class (in model units)
K_sat = {
    10: 1.8e-6,   # Tree cover    — 6.48 mm/h
    20: 3.0e-6,   # Shrubland     — 10.92 mm/h
    30: 9.4e-7,   # Grassland     — 3.38 mm/h
    40: 9.4e-7,   # Cropland      — 3.38 mm/h
    50: 1.0e-8,   # Built-up      — impervious
    60: 8.3e-6,   # Bare/sparse   — 29.97 mm/h (Rawls)
    80: 0.0,      # Water bodies  — saturated
    90: 3.0e-7,   # Wetland       — 1.08 mm/h
}

psi = {
    10: 0.167, 20: 0.110, 30: 0.089, 40: 0.089,
    50: 0.050, 60: 0.061, 80: 0.0,   90: 0.316,
}

delta_theta = {
    10: 0.486, 20: 0.412, 30: 0.434, 40: 0.434,
    50: 0.050, 60: 0.401, 80: 0.0,   90: 0.385,
}

K_array      = make_grid(K_sat)
psi_array    = make_grid(psi)
dtheta_array = make_grid(delta_theta)

case_input.set_grid_parameter(hydraulic_conductivity=K_array)
case_input.set_grid_parameter(capillary_head=psi_array)
case_input.set_grid_parameter(water_content_diff=dtheta_array)
case_input.set_grid_parameter(cumulative_depth=0.0)  # dry soil
```
