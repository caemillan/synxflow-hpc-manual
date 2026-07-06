# Módulo de Infiltración Green-Ampt

## Fundamento teórico

El módulo de infiltración de SynxFlow se basa en el modelo de Green-Ampt (1911), una solución analítica derivada de la ley de Darcy para un frente de humectación tipo pistón. El modelo asume que el suelo por encima del frente está completamente saturado y el suelo por debajo mantiene su contenido de humedad inicial.

La tasa de infiltración instantánea $f(t)$ [m/s] se expresa como:

$$f(t) = K_s \left(1 + \frac{\psi \cdot \Delta\theta}{F(t)}\right)$$

Donde:

| Símbolo                              | Parámetro SynxFlow       | Descripción                                  | Unidades |
| ------------------------------------ | ------------------------ | -------------------------------------------- | -------- |
| $K_s$                                | `hydraulic_conductivity` | Conductividad hidráulica saturada efectiva   | m/s      |
| $\psi$                               | `capillary_head`         | Succión capilar promedio en el frente húmedo | m        |
| $\Delta\theta = \theta_s - \theta_i$ | `water_content_diff`     | Déficit de humedad (porosidad "llenable")    | —        |
| $F(t)$                               | `cumulative_depth`       | Infiltración acumulada                       | m        |

A medida que $F(t)$ crece, la tasa de infiltración decrece asintóticamente hacia $K_s$. En el largo plazo (suelo saturado), toda la capacidad de infiltración está dada por $K_s$.

## Implementación en GPU (CUDA)

En SynxFlow, la infiltración se calcula en cada celda activa del dominio y en cada paso de tiempo del solucionador. La solución analítica para un paso de tiempo $\Delta t$ se obtiene resolviendo implícitamente la ODE $dF/dt = f(F)$:

$$F_1 = \frac{1}{2}\left[(F_0 + K_s \Delta t) + \sqrt{(F_0 + K_s \Delta t)^2 + 4\,K_s\,\Delta t\,(\psi + h)\,\Delta\theta}\right]$$

Donde $F_0$ es la infiltración acumulada al inicio del paso, $h$ es la lámina de agua disponible y $F_1$ es la infiltración acumulada al final del paso.

La variación efectiva de infiltración en el paso $\Delta t$ se limita por la lámina de agua disponible:

$$\Delta F = \min(h,\ F_1 - F_0)$$

El solucionador actualiza en cada celda:

```
cumulative_depth += ΔF
h               -= ΔF
```

Esta formulación garantiza que la celda no infiltre más agua de la que contiene, evitando profundidades negativas.

### Kernel CUDA

El kernel GPU que implementa esta lógica opera sobre todas las celdas activas en paralelo:

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

El kernel se invoca desde el módulo de detritos después de cada integración temporal de las SWE.

## Parámetros por clase de cobertura (Rawls et al., 1983)

Los parámetros de Green-Ampt se asignan espacialmente,para efectos de este manual, según la clasificación de cobertura de suelo ESA WorldCover (10 m), remuestreada al DEM de simulación. Los valores de referencia provienen de Rawls et al. (1983), Tabla 4.5.2, obtenidos para suelos de zonas templadas de EE.UU.

| Clase ESA | Cobertura    | Tipo de suelo                 | $K_s$ [mm/h] | $\psi$ [cm] | $\Delta\theta$ [-] |
| --------- | ------------ | ----------------------------- | ------------ | ----------- | ------------------ |
| 10        | Tree cover   | Franco limoso _(Silt loam)_   | 6.48         | 16.68       | 0.486              |
| 20        | Shrubland    | Franco arenoso _(Sandy loam)_ | 10.92        | 11.01       | 0.412              |
| 30        | Grassland    | Franco _(Loam)_               | 3.38         | 8.89        | 0.434              |
| 40        | Cropland     | Franco _(Loam)_               | 3.38         | 8.89        | 0.434              |
| 50        | Built-up     | Impermeable                   | 0.036        | 5.00        | 0.050              |
| 60        | Bare/sparse  | Arena limosa _(Loamy sand)_   | 29.97        | 6.13        | 0.401              |
| 80        | Water bodies | Saturado                      | 0.0          | 0.0         | 0.000              |
| 90        | Wetland      | Arcilloso _(Clay)_            | 1.08         | 31.63       | 0.385              |

> **Nota:** Los valores en unidades del modelo (m/s y m) se obtienen como:\
> $K_s\ [\text{m/s}] = K_s\ [\text{mm/h}] \times 2.778 \times 10^{-7}$\
> $\psi\ [\text{m}] = \psi\ [\text{cm}] \times 0.01$

## Consideraciones para cuencas andinas semi-áridas

Los valores de Rawls et al. (1983) fueron obtenidos en suelos agrícolas de zonas templadas con perfil completo. En cuencas andinas semi-áridas (e.g., cuenca Rímac alta, Chosica), se aplican factores de reducción por las siguientes razones físicas:

### Sellamiento superficial (_rain crust_)

El impacto cinético de las gotas de lluvia disgrega los agregados del suelo, compacta la superficie y obstruye los poros macroscópicos (_structural sealing_). En suelos con material de grano fino (limo, ceniza volcánica, material coluvial), las partículas migran hacia el interior y sellan los poros superficiales (_depositional sealing_). Ambos procesos reducen $K_s$ superficial en uno a dos órdenes de magnitud respecto al valor de laboratorio (Assouline, 2004).

La formación de costra por lluvia tiene consecuencias directas en la generación de flujos:

- **Generación rápida de escorrentía**: al suprimir la infiltración en los primeros minutos del evento, la costra desvía la precipitación directamente a flujo superficial, reduciendo la lámina necesaria para generar escorrentía (Assouline, 2004).
- **Aumento del caudal pico**: la costra actúa como superficie casi impermeable, incrementando el coeficiente de escorrentía y la probabilidad de iniciación de flujos hiperconcentrados y de detritos (Thouret et al., 2020, _Earth-Sci. Rev._ 201:103003).
- **Costras endurecidas por depósitos previos**: en quebradas con recurrencia de eventos, los depósitos de flujos anteriores forman costras cementadas que suprimen la infiltración en el fondo del canal, facilitando la avulsión hacia canales secundarios (Thouret et al., 2020).

El proceso es especialmente relevante en la clase 60 (suelo desnudo / roca) al inicio del evento Yaku 2023, cuando el suelo se encontraba en condición seca previo inicio de las lluvias.

### Hidrofobicidad en suelos secos

Los suelos de zonas semi-áridas presentan repelencia al agua (_hydrophobicity_) cuando están secos, reduciendo la infiltración efectiva durante las primeras horas de lluvia. La repelencia disminuye a medida que el suelo se humedece (umbrales reportados por Doerr et al., 2000). En entornos volcánicos y quebradas andinas con vegetación perennifolia, la hidrofobicidad explica la alta frecuencia de lahares y flujos de detritos ante lluvias de baja intensidad al inicio de la estación húmeda, cuando el suelo seco repele el agua generando escorrentía superficial inmediata (Capra et al., 2010, _J. Volcanol. Geotherm. Res._ 189:105–117).

### Suelos delgados sobre roca

En la cuenca alta del Rímac (por encima de 2000 m.s.n.m.), la clase 60 cubre más del 60% del área y corresponde a suelos delgados (<20 cm) sobre roca granítica o metamórfica. La infiltración efectiva está limitada por la capacidad de almacenamiento del perfil, no por $K_s$.

### Factores de reducción recomendados

Para la cuenca Rímac (evento Yaku 2023), el análisis de sensibilidad OAT (_One-At-a-Time_: se varía un parámetro a la vez manteniendo los demás fijos) sobre $K_s$ de la clase 60 sugiere:

| Multiplicador | $K_s$ clase 60 [mm/h] | Justificación                        |
| ------------- | --------------------- | ------------------------------------ |
| ×1.00 (Rawls) | 29.97                 | Suelo no perturbado, referencia      |
| ×0.50         | 14.94                 | Suelo con moderado sellamiento       |
| ×0.18         | 5.39                  | Suelo andino rocoso, costra severa   |
| ×0.10         | 2.99                  | Sellamiento extremo / roca aflorante |
| ×0.05         | 1.49                  | Cubierta rocosa >80%                 |

> La sensibilidad de la respuesta hidrométrica al multiplicador de $K_s$ clase 60 es baja en el canal principal del Rímac (~4% de variación en $h_{max}$), indicando que la condición de borde aguas arriba domina sobre la infiltración en el tramo bajo de la cuenca.

## Variable de estado `cumulative_depth`

`cumulative_depth` ($F$) representa la infiltración acumulada desde el inicio de la simulación [m]. Es una **variable de estado** con significado físico:

- $F = 0$: suelo completamente seco (condición inicial estándar).
- $F > 0$: suelo con humedad antecedente. La tasa de infiltración inicial es menor que con $F=0$.
- $F \to \infty$: suelo saturado; $f \to K_s$ (estado estacionario).

### Inicialización con humedad antecedente

Para eventos precedidos por lluvias previas, la condición inicial $F=0$ subestima la humedad antecedente y sobreestima la capacidad de infiltración. En el caso de Yaku 2023, el suelo se encontraba en condición relativamente seca antes del inicio de las lluvias, por lo que $F=0$ es una aproximación razonable para ese evento. Se recomienda ajustar este valor cuando existan registros o estimaciones de lluvia antecedente significativa:

```python
# Suelo seco (por defecto)
case_input.set_grid_parameter(cumulative_depth=0.0)

# Suelo pre-húmedo (equivalente a F antecedente estimado)
F_antecedente = 0.02  # m, equivale a ~20 mm de lluvia previa
case_input.set_grid_parameter(cumulative_depth=F_antecedente)
```

### Salidas de `cumulative_depth`

SynxFlow escribe `cumulative_depth` como:

- Raster periódico: `cumulative_depth_{t}.asc` (cada intervalo de salida).
- Archivo de respaldo: `cumulative_depth_backup_{t}.dat` (cada intervalo de backup).

Esto permite retomar simulaciones en cascada y analizar la distribución espacial de la infiltración acumulada al final del evento.

## Asignación de parámetros en Python

```python
# Parámetros Green-Ampt por clase ESA (en unidades del modelo)
K_sat = {
    10: 1.8e-6,   # Tree cover    — 6.48 mm/h
    20: 3.0e-6,   # Shrubland     — 10.92 mm/h
    30: 9.4e-7,   # Grassland     — 3.38 mm/h
    40: 9.4e-7,   # Cropland      — 3.38 mm/h
    50: 1.0e-8,   # Built-up      — impermeable
    60: 8.3e-6,   # Bare/sparse   — 29.97 mm/h (Rawls)
    80: 0.0,      # Water bodies  — saturado
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
case_input.set_grid_parameter(cumulative_depth=0.0)  # suelo seco
```
