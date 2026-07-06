# Debris Flow Module

## Flow classification by sediment concentration

Several approaches exist for classifying and typifying hillslope flows; among the best known are Costa (1988), Coussot (1997), Suárez (2001), and the USGS and Flo-2D software manual classifications. For the purposes of analysing flow types in this region, the Costa (1988) classification was adopted, as it is considered to best explain the local flow dynamics. Hillslope and gully flows are classified according to their volumetric sediment concentration $C_s$ [-]. Costa (1988) defines three regimes with thresholds based on the rheological behaviour of the mixture:

| Flow type | $C_s$ [vol.] | Rheological behaviour |
|-----------|-------------|----------------------|
| **Streamflow** | $< 0.20$ | Newtonian — bed load and suspension transport |
| **Hyperconcentrated flow** | $0.20 - 0.47$ | Incipient viscoplastic — measurable yield strength |
| **Debris flow** | $0.47 - 0.77$ | Visco-plastic — grain collisions dominate |
| **Landslide** | $> 0.77$ | Mass movement — solid behaviour |

> The boundaries between classes may overlap, especially in the range $0.20 < C_s < 0.47$ where the transition is progressive and depends on grain size and the density of the interstitial fluid (Costa, 1988).

In SynxFlow, $C_s$ is calculated in each cell and at each time step as a state variable. The output `C_{t}.asc` allows spatial classification of the flow type throughout the event.

## Comparison of debris flow models

Numerical debris flow models differ primarily in their rheological formulation of basal stress $\tau$. The table below compares the most widely used models in professional and research practice:

| Model | Rheology | Basal stress equation | Key parameters | Numerical method | Solver | Reference |
| ----- | -------- | --------------------- | -------------- | ---------------- | ------ | --------- |
| **SynxFlow** | Manning + Mohr-Coulomb | $\tau = \mu_s(\rho_m-\rho_w)gh\cos\theta + \mu_d\rho_m gh + \rho_m gn^2 u\|u\|h^{-1/3}$ | $n$, $\mu_s$, $\mu_d$, $\alpha$, $\beta$ | Finite volume (Godunov) | HLLC Riemann — GPU (CUDA) | Xia et al. (2023), §2–3 |
| **Flo-2D** | Quadratic (O'Brien) | $\tau = \tau_y + \eta \dfrac{du}{dz} + \rho_m K \left(\dfrac{du}{dz}\right)^2$ | $\tau_y$, $\eta$, $K$, $n$ | Finite difference | Explicit (upwind) | O'Brien et al. (1993) |
| **RAMMS::DEBRIS FLOW** | Voellmy | $\tau = \mu\rho_m gh\cos\theta + \dfrac{\rho_m gu^2}{\xi}$ | $\mu$, $\xi$ [m/s²] | Finite volume | Godunov-type | Christen et al. (2010) |
| **RiverFlow2D** | Multiple: Manning, Coulomb, Bingham, quadratic | 8 formulations (Table 1); internal and basal friction from clean water to hyperconcentrated mixture | $n$, $\phi_b$, $\tau_y$, $\eta$, $K$ | Finite volume | Augmented Roe Riemann — GPU | Murillo & García-Navarro (2012) |
| **EDDA 2.0** | Two-phase: Coulomb (solid) + Manning (fluid) | $\tau = (\rho_s-\rho_f)gh\tan\phi_{bed} + \rho_f gn^2(u^2+v^2)h^{-1/3}$ | $n$, $\phi_{bed}$, $c_0$, $\phi_{bin}$, $\lambda$ (pore pressure) | Finite difference | MacCormack-TVD | Shen et al. (2018) |
| **DAN3D / DAN-W** | Variable (Coulomb, Voellmy, Bingham) | User-selectable per event | $\phi_b$, $\mu$, $\xi$, $\tau_y$ | Lagrangian (SPH-type) | Lagrangian continuum | McDougall & Hungr (2004) |
| **TITAN2D** | Coulomb (mixture) | $\tau = (p - p_f)\tan\phi_b$ | $\phi_b$, $\phi_{int}$ | Finite volume | Godunov-type | Pitman et al. (2003) |
| **D-Claw (USGS)** | Coulomb + pore pressure | $\tau = (\sigma - p_f)\tan\phi$ | $\phi$, $p_f$, dilatancy $\psi$ | Finite volume | Godunov-type (GeoClaw) | George & Iverson (2014) |

### Notes on model selection

- **SynxFlow** and **RiverFlow2D** are the only GPU-native models in the table, enabling domains with millions of cells with reduced computation times. RiverFlow2D uses an augmented Roe Riemann solver with 8 rheological formulations (Murillo & García-Navarro, 2012, *J. Comput. Phys.* 231:1963–2001); its main limitation for operational use is that it is commercial software.
- **EDDA 2.0** couples two geotechnical initiation mechanisms (slope failure by soil saturation + surface runoff) with flow propagation. Unlike SynxFlow — which generates sediment through hydraulic bed erosion as flow exceeds the critical shear stress — EDDA 2.0 includes explicit geotechnical failure criteria (pore pressure, cohesion, internal friction angle) as the mass initiation mechanism (Shen et al., 2018, *Geosci. Model Dev.* 11:2841–2856).
- **Flo-2D** is widely used in hazard studies for its commercial interface and quadratic rheology for hyperconcentrated flows; it does not model dynamic bed erosion.
- **RAMMS** is the de facto standard for back-analysis in the Alps and Andes; its strength is the two-parameter Voellmy parameterisation ($\mu$, $\xi$).
- **DAN3D** is Lagrangian (no fixed mesh), useful for long-runout flows; it allows changing rheology along the flow path.
- **D-Claw** is physically the most complete model as it solves pore pressure as a state variable, but requires detailed geotechnical input data.

### Rationale for using SynxFlow in the Rímac EWS

SynxFlow was selected as the simulation engine for the EWS for the following reasons:

- **Open-source** (GPL-3.0 licence): permits modification, integration, and distribution without commercial restrictions. RiverFlow2D, with comparable GPU capabilities, is paid software.
- **Python/HPC integration**: the native Python interface enables automation of the full rainfall → simulation → postprocessing cycle in SENAMHI's cluster environment.
- **Complete flow chain in a single solver**: integrates rainfall → infiltration (Green-Ampt) → runoff → hydraulic bed erosion → water-sediment mixture propagation, all within a single GPU kernel with no inter-module data transfer.
- **Scalability**: with 1 H100 GPU, the 8.2-million active-cell domain at 5 m resolution is simulated in 20 min–7 h depending on event intensity, consistent with EWS operational time requirements.

## General description

The SynxFlow debris flow module solves the shallow water equations (SWE) extended for water-solid mixtures, including Coulomb friction terms, dynamic erosion/deposition, and infiltration. The solver is implemented in CUDA C++ and operates entirely on GPU, enabling simulation of domains with millions of cells in computation times on the order of hours.

Unlike the flood module (*flood*), the debris module (*debris*) incorporates:

- Volumetric sediment concentration **C** as an additional state variable.
- Coulomb friction dependent on mixture density.
- Mass exchange between the flow and the bed (erosion and deposition).
- Green-Ampt infiltration coupled directly to the volume balance.

## Governing equations

The module solves the conservation equations for mass and momentum for a water-solid mixture:

### Mixture volume conservation

$$\frac{\partial h}{\partial t} + \frac{\partial (hu)}{\partial x} + \frac{\partial (hv)}{\partial y} = S_v - f$$

Where $h$ is the mixture depth [m], $u$ and $v$ are the velocity components [m/s], $S_v$ is the volumetric source term (rainfall, inflows), and $f$ is the Green-Ampt infiltration rate [m/s].

### Momentum conservation

$$\frac{\partial (hu)}{\partial t} + \frac{\partial}{\partial x}\left(hu^2 + \frac{g h^2}{2}\right) + \frac{\partial (huv)}{\partial y} = -gh\frac{\partial z}{\partial x} - \frac{\tau_x}{\rho_m}$$

$$\frac{\partial (hv)}{\partial t} + \frac{\partial (huv)}{\partial x} + \frac{\partial}{\partial y}\left(hv^2 + \frac{g h^2}{2}\right) = -gh\frac{\partial z}{\partial y} - \frac{\tau_y}{\rho_m}$$

Where $z$ is the bed elevation [m], $g$ is the gravitational acceleration [m/s²], $\rho_m$ is the mixture density [kg/m³], and $\boldsymbol{\tau}$ is the total bed stress.

### Sediment conservation

$$\frac{\partial (hC)}{\partial t} + \frac{\partial (huC)}{\partial x} + \frac{\partial (hvC)}{\partial y} = E - D$$

Where $C$ is the volumetric sediment concentration [-], $E$ is the erosion rate, and $D$ is the deposition rate [m/s].

### Mixture density

$$\rho_m = \rho_w (1 - C) + \rho_s C$$

With $\rho_w = 1000$ kg/m³ (water) and $\rho_s \approx 2650$ kg/m³ (solids).

## Coulomb friction model

The bed stress in the debris module is modelled using Coulomb friction, appropriate for dense granular flows:

$$\tau = \mu_s \cdot (\rho_m - \rho_w) \cdot g \cdot h \cdot \cos\theta + \mu_d \cdot \rho_m \cdot g \cdot h$$

Where:

| Symbol | SynxFlow parameter | Description | Typical value |
|--------|--------------------|-------------|---------------|
| $\mu_s$ | `static_friction_coeff` | Static friction coefficient (tangent of the angle of repose) | 0.62 |
| $\mu_d$ | `dynamic_friction_coeff` | Dynamic friction coefficient | 0.44 |
| $\theta$ | — | Local bed inclination angle | from DEM |

The reference values ($\mu_s = 0.62$, $\mu_d = 0.44$) were determined by Xia et al. (2023) through calibration against the flume experiment of Takahashi et al. (1992), and serve as the starting point for calibration in the Rímac watershed.

### Back-analysis values from arid and semi-arid Andean watersheds

The following Coulomb friction values have been determined by back-analysis at Andean sites under conditions similar to the Rímac watershed, and serve as an orientation range for calibration:

| Site | Country | Condition | Model | $\mu$ | Reference | Page |
| ---- | ------- | --------- | ----- | ----- | --------- | ---- |
| Takahashi flume | Lab. | Flume experiment | SynxFlow | $\mu_s=0.62$, $\mu_d=0.44$ | Xia et al. (2023), *Eng. Geol.* 326:107310 | §3 (p. 6) |
| Quebrada Sahuanay, Abancay | Peru | Dry phase | RAMMS | $\mu=0.20$ | Rodríguez-Morata et al. (2019), *Geomorphology* 342:127–139 | p. 133 |
| Quebrada Sahuanay, Abancay | Peru | Wet phase | RAMMS | $\mu=0.10$ | Rodríguez-Morata et al. (2019), *Geomorphology* 342:127–139 | p. 133 |
| Lake 513, Carhuaz (granular GLOF) | Peru | Granular flow | RAMMS | $\mu=0.16$ | Schneider et al. (2014), *Adv. Geosci.* 35:145–155 | p. 149 |
| Lake 513, Carhuaz (viscous GLOF) | Peru | Viscous flow | RAMMS | $\mu=0.08$ | Schneider et al. (2014), *Adv. Geosci.* 35:145–155 | p. 149 |
| Lake 513, Carhuaz (hyperconc.) | Peru | Hyperconcentrated | RAMMS | $\mu=0.04$ | Schneider et al. (2014), *Adv. Geosci.* 35:145–155 | p. 149 |
| Manflas, arid Andes | Chile | GLOF — long runout | RAMMS | $\mu \leq 0.001$ | Iribarren Anacona et al. (2018), *Nat. Hazards* 94:93–119 | p. 104 |

> **Note:** The Sahuanay, Carhuaz and Manflas values were extracted from the review table of Mikoš & Bezak (2021), pp. 2, 5 and 6 respectively, which consolidates more than 30 RAMMS back-analyses across different mountain environments. RAMMS $\mu$ values (Voellmy friction) are not directly equivalent to SynxFlow's $\mu_s$ / $\mu_d$ (decoupled Coulomb), but are indicative of the expected range for these materials.

## Erosion and deposition

### Erodible depth

The bed available for erosion is limited by the `erodible_depth` parameter [m], which represents the maximum thickness of material that can be removed. This value varies by land cover class:

| ESA WorldCover class | Land cover | `erodible_depth` [m] |
|---------------------|-----------|---------------------|
| 10 — Tree cover | Forest | 0.5 |
| 20 — Shrubland | Shrubland | 0.3 |
| 30 — Grassland | Grassland | 0.3 |
| 40 — Cropland | Cropland | 0.4 |
| 50 — Built-up | Urban area | 0.0 |
| 60 — Bare/sparse | Bare soil / rock | 0.8 |
| 80 — Water bodies | Water bodies | 0.0 |
| 90 — Wetland | Wetland | 0.1 |

Class 60 (bare soil/rock, dominant in the upper Andean catchment) has the highest value due to the availability of unconsolidated colluvial material.

### State variable `erodible_depth`

`erodible_depth` is a variable that **changes during the simulation**: it decreases as material is eroded and increases through deposition. It is written as a periodic output and as a backup file, which allows:

- Inspecting the spatial distribution of cumulative erosion.
- Resuming a simulation from an intermediate state with the correct bed.

## Initial concentration

The volumetric sediment concentration is initialised as zero (`C = 0`) at the start of the simulation, representing clean water. The model progressively generates sediment as the flow erodes the bed.

The upstream boundary condition can specify an inlet concentration (`C_in`), useful for representing flows with sediment load from sub-catchments not explicitly modelled.

## Debris module output variables

| File | Variable | Units | Description |
|------|----------|-------|-------------|
| `h_{t}.asc` | $h$ | m | Mixture depth |
| `hUx_{t}.asc` | $hu$ | m²/s | Unit discharge in X |
| `hUy_{t}.asc` | $hv$ | m²/s | Unit discharge in Y |
| `cumulative_depth_{t}.asc` | $F$ | m | Cumulative infiltration |
| `erodible_depth_{t}.asc` | $d_e$ | m | Remaining erodible depth |
| `h_gauges.dat` | $h(t)$ | m | Time series at virtual gauges |

Mixture velocity is calculated in postprocessing as:

$$v = \frac{\sqrt{(hu)^2 + (hv)^2}}{h}$$

## Python configuration

```python
from synxflow import IO, debris

# Rheological parameters (uniform across domain)
static_friction  = np.full(DEM.array.shape, 0.62)
dynamic_friction = np.full(DEM.array.shape, 0.44)
concentration    = np.zeros(DEM.array.shape)   # initially clean water

# Erodible depth by ESA class
ero_depth = {10: 0.5, 20: 0.3, 30: 0.3, 40: 0.4,
             50: 0.0, 60: 0.8, 80: 0.0, 90: 0.1}
ero_array = make_grid(ero_depth)

# Parameter assignment to model
case_input.add_user_defined_parameter('C',                      concentration)
case_input.add_user_defined_parameter('erodible_depth',         ero_array)
case_input.add_user_defined_parameter('static_friction_coeff',  static_friction)
case_input.add_user_defined_parameter('dynamic_friction_coeff', dynamic_friction)

# GPU execution (1 GPU)
debris.run(case_folder)

# Multi-GPU execution
debris.run_mgpus(case_folder)
```

## Considerations for Andean watersheds

In mountain catchments of the gully type (slopes >5%, colluvial material), the following is recommended:

- **`static_friction_coeff`**: calibrate in the range 0.55–0.70 depending on material grain size.
- **`dynamic_friction_coeff`**: typically 0.65–0.80 of the static value.
- **`erodible_depth`**: verify against post-event geomorphological surveys.
- The output concentration `C` can be compared against sediment load samples at the hydrometric station.
