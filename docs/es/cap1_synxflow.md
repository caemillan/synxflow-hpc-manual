# SynxFlow

## Descripción del modelo

SynxFlow (_Synergising High-Performance Hazard Simulation with Data Flow_) es un modelo de código abierto capaz de simular dinámicamente inundaciones, deslizamientos de tierra y flujos de detritos mediante GPU compatibles con CUDA. Cuenta con una interfaz Python intuitiva y versátil, y se integra a la perfección con herramientas como NumPy, Pandas y GDAL para un procesamiento eficiente de los datos de entrada, la ejecución de la simulación y la visualización de los resultados. Gracias a la aceleración de las GPU modernas, SynxFlow puede completar simulaciones a gran escala en cuestión de minutos, lo que lo convierte en una solución ideal para integrarse en flujos de trabajo de ciencia de datos y optimizar y acelerar la evaluación de riesgos de desastres naturales, potenciando tanto la investigación como las aplicaciones empresariales (Repositorio de SynxFlow, 2025).

Los autores de SynxFlow cuentan con amplia experiencia en el desarrollo y la aplicación de modelos hidrodinámicos. Los algoritmos empleados en el modelo son altamente eficientes y robustos, y utilizan métodos de captura de choques tipo Godunov para resolver ecuaciones en aguas poco profundas (SWE: _Shallow Water Equations_). Los métodos numéricos del modelo se documentan en los artículos de Xia et al. (2017, 2023) y Xia & Liang (2018).

### Ecuaciones de gobierno (SWE)

La modelación hidrodinámica se implementa bajo un enfoque bidimensional, considerando la necesidad de una representación precisa de la dinámica del flujo. Bajo este enfoque, se resuelven las ecuaciones de aguas poco profundas (_Shallow-Water Equations_, SWE) (Vreugdenhil, 2013), adecuadas para la descripción del movimiento de fluidos poco profundos.

El modelo SWE resuelve las ecuaciones de conservación de volumen y momento, además de incluir aceleración temporal y espacial.

**Conservación de volumen:**

$$\frac{\partial h}{\partial t} + \frac{\partial (hu)}{\partial x} + \frac{\partial (hv)}{\partial y} = S_v$$

**Conservación de momento en X:**

$$\frac{\partial (hu)}{\partial t} + \frac{\partial}{\partial x}\left(hu^2 + \frac{gh^2}{2}\right) + \frac{\partial (huv)}{\partial y} = -gh\frac{\partial z}{\partial x} - \frac{\tau_x}{\rho}$$

**Conservación de momento en Y:**

$$\frac{\partial (hv)}{\partial t} + \frac{\partial (huv)}{\partial x} + \frac{\partial}{\partial y}\left(hv^2 + \frac{gh^2}{2}\right) = -gh\frac{\partial z}{\partial y} - \frac{\tau_y}{\rho}$$

Donde:

| Símbolo        | Descripción                                      | Unidades |
| -------------- | ------------------------------------------------ | -------- |
| $\eta = z + h$ | Elevación de la superficie del flujo             | m        |
| $t$            | Tiempo                                           | s        |
| $h$            | Profundidad de flujo (calado)                    | m        |
| $u, v$         | Componentes de velocidad en X e Y                | m/s      |
| $S_v$          | Término fuente o sumidero (lluvia, infiltración) | m/s      |
| $g$            | Aceleración gravitacional (9.81)                 | m/s²     |
| $\nu_t$        | Viscosidad turbulenta de remolino                | m²/s     |
| $\tau$         | Estrés basal total                               | N/m²     |
| $\rho$         | Densidad de la mezcla agua-sólido                | kg/m³    |
| $R$            | Radio hidráulico                                 | m        |
| $S_f$          | Pendiente de la superficie de flujo              | —        |
| $\theta$       | Ángulo de inclinación del vector de velocidad    | rad      |

En SynxFlow (Xia et al., 2017), las ecuaciones SWE se resuelven con algoritmos eficientes y robustos con métodos de captura de choques de tipo Godunov (Toro, 2001). El **solucionador de Riemann HLLC** (_Harten-Lax-van Leer-Contact_) garantiza la captura correcta de discontinuidades hidráulicas (frentes de onda, resaltos) que ocurren en eventos extremos con múltiples hidrógrafos de entrada y confluencias. La integración temporal utiliza el esquema de Euler explícito con control adaptativo del paso de tiempo mediante la condición CFL (_Courant-Friedrichs-Lewy_).

### Coeficiente de rugosidad de Manning

La resistencia al flujo se parametriza mediante el coeficiente de rugosidad de Manning $n$ [m⁻¹/³·s], asignado espacialmente, para el caso del presente manual, según la clasificación de cobertura de suelo ESA WorldCover v200 (10 m, remuestreada a 5 m). Los valores utilizados en la simulación del evento Yaku 2023 (cuenca del río Rímac) son:

| Clase ESA | Cobertura                          | Manning $n$ [m⁻¹/³·s] |
| --------- | ---------------------------------- | --------------------- |
| 10        | Tree cover (bosque)                | 0.100                 |
| 20        | Shrubland (matorral)               | 0.050                 |
| 30        | Grassland (pastizal)               | 0.040                 |
| 40        | Cropland (cultivos)                | 0.030                 |
| 50        | Built-up (zona urbana)             | 0.020                 |
| 60        | Bare/sparse (suelo desnudo / roca) | 0.030                 |
| 70        | Snow and ice                       | 0.000                 |
| 80        | Water bodies (cuerpos de agua)     | 0.030                 |
| 90        | Herbaceous wetland (humedal)       | 0.040                 |
| 100       | Moss and lichen                    | 0.010                 |
| —         | Default (clases no asignadas)      | 0.035                 |

Los valores se asignan al modelo mediante `set_grid_parameter`:

```python
case_input.set_landcover(landcover)
case_input.set_grid_parameter(manning={
    'param_value':   [0.10, 0.05, 0.04, 0.03, 0.02, 0.03, 0.0, 0.03, 0.04, 0.0, 0.01],
    'land_value':    [  10,   20,   30,   40,   50,   60,  70,   80,   90,  95, 100],
    'default_value': 0.035})
```

## Metodología

### Enfoque de modelamiento

SynxFlow implementa un enfoque de modelamiento bidimensional en malla estructurada regular (_raster_), donde cada celda del DEM corresponde a una celda computacional. El modelo resuelve las SWE en cada celda activa del dominio en cada paso de tiempo, calculando los flujos interceldas mediante el solucionador de Riemann HLLC.

Los módulos disponibles son:

| Módulo                       | Función Python    | Descripción                                   |
| ---------------------------- | ----------------- | --------------------------------------------- |
| Inundación (_flood_)         | `flood.run()`     | SWE + lluvia + infiltración GA                |
| Flujo de detritos (_debris_) | `debris.run()`    | SWE + Coulomb + erosión _+ infiltración GA_\* |
| Deslizamiento (_landslide_)  | `landslide.run()` | Movimiento de masa granular                   |

\* El modulo de infiltración GA fue implementado al solver de _debris_ como parte de la implementación del SAT.

### Dominio del modelo y mallado

El dominio de simulación se define a partir de un Modelo Digital de Elevación (DEM) en formato GeoTIFF o ASC. SynxFlow utiliza directamente la resolución y extensión del DEM como grilla computacional.

**Consideraciones para el mallado:**

- **Resolución**: determina el tamaño de las celdas y el costo computacional. A 5 m de resolución, la cuenca del Rímac (Chosica) genera ~8.2 millones de celdas activas.
- **Celdas NODATA**: las celdas fuera de la cuenca se excluyen automáticamente del cómputo.
- **DEM preprocesado**: se recomienda rellenar depresiones (_pit filling_) y suavizar artefactos antes de la simulación para evitar inestabilidades numéricas.

```python
from synxflow import IO

DEM = IO.Raster('DEM_5m_filled.tif')
case_input = IO.InputModel(DEM, num_of_sections=1,
                           case_folder='/scratch/.../caso/')
```

### Parametrización de los modelos

Los modelos requieren los siguientes datos de entrada:

| Dato                       | Formato       | Descripción                               |
| -------------------------- | ------------- | ----------------------------------------- |
| DEM                        | GeoTIFF / ASC | Modelo Digital de Elevación preprocesado  |
| Cobertura de suelo         | GeoTIFF       | Clasificación para Manning e infiltración |
| Máscara de lluvia          | GeoTIFF       | Extensión espacial de la lluvia           |
| Serie temporal de lluvia   | CSV           | Intensidad por zona [m/s] vs tiempo [s]   |
| Condiciones de contorno    | Python dict   | Hidrógramas de entrada y salida           |
| Parámetros de infiltración | numpy array   | Green-Ampt por celda                      |
| Parámetros de detritos     | numpy array   | Fricción y profundidad erosionable        |

## Caso de aplicación: evento Yaku 2023 — cuenca del río Rímac

El módulo de flujo de detritos (_debris_) fue aplicado a la cuenca media-baja del río Rímac (Chosica, Lima) para modelar el evento Yaku ocurrido entre el 9 y 16 de marzo de 2023. Este evento produjo flujos de detritos e inundaciones simultáneos en múltiples quebradas tributarias (Huascarán, Cusipata, Los Cóndores), con pérdidas materiales severas en la infraestructura vial y residencial de Chosica.

La simulación se configuró con:

- **DEM**: SPOT-6 a 5 m de resolución (EPSG:32718), relleno de depresiones previo.
- **Dominio**: ~17.5 millones de celdas totales, ~8.2 millones activas (47%).
- **Lluvia**: pronóstico WRF a 1 km, desacumulado y reproyectado desde EPSG:4326.
- **Duración**: 192 horas (9 días), salida horaria.
- **Condición de borde**: hidrógrafo aguas arriba en el río Rímac + hidrógramas de lluvia en quebradas.

La simulación se utilizó como back-analysis (retroanálisis) para la calibración y validación del SAT, comparando la profundidad simulada con registros en gauges virtuales ubicados en las quebradas tributarias y en el cauce principal.

## Sensibilidad paramétrica del módulo Green-Ampt

La cuenca alta del Rímac presenta predominio de suelo desnudo sobre roca (clase ESA 60, >60% del área sobre 2000 m.s.n.m.), con alta susceptibilidad a la erosión y generación rápida de escorrentía. El análisis de sensibilidad OAT (_One-At-a-Time_) sobre la conductividad hidráulica saturada $K_s$ de la clase 60 mostró que:

- La variación de $K_s$ entre ×0.05 y ×1.00 (respecto a Rawls et al., 1983) produce ~4% de variación en $h_{max}$ en el canal principal del Rímac.
- La respuesta hidrométrica está dominada por la condición de borde aguas arriba, no por la infiltración en el tramo bajo.
- Para cuencas andinas semi-áridas se recomienda aplicar factores de reducción de $K_s$ entre 0.10 y 0.30 respecto a Rawls, para representar el sellamiento superficial y la hidrofobicidad en suelos secos.

## Glosario de términos

| Término        | Descripción                                                                        |
| -------------- | ---------------------------------------------------------------------------------- |
| **SWE**        | _Shallow Water Equations_ — ecuaciones de aguas poco profundas 2D                  |
| **DEM**        | _Digital Elevation Model_ — Modelo Digital de Elevación                            |
| **GPU**        | _Graphics Processing Unit_ — unidad de procesamiento gráfico para cómputo paralelo |
| **HPC**        | _High-Performance Computing_ — computación de alto rendimiento (clúster)           |
| **CFL**        | Condición de estabilidad de Courant-Friedrichs-Lewy                                |
| **HLLC**       | Solucionador de Riemann Harten-Lax-van Leer-Contact                                |
| **Green-Ampt** | Modelo de infiltración con frente de humectación tipo pistón                       |
| **Manning**    | Coeficiente de rugosidad hidráulica empírico                                       |
| **Solver**     | Módulo de cómputo en GPU que resuelve las SWE                                      |
| **Gauge**      | Punto virtual de medición dentro del dominio de simulación                         |
| **Backup**     | Archivo de respaldo del estado del modelo en un instante dado                      |
