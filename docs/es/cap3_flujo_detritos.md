# Módulo de Flujo de Detritos

## Clasificación de flujos por concentración de sedimentos

Existen diferentes enfoques para clasifica y tipificar los flujos en laderas, sin embargo, entre los más conocidos se encuentran Costa (1988), Coussot (1997), Suaréz (2001), incluso la USGS y el manual de software Flo-2D. Para los fines de analisis de los tipos flujos para esta zona, se optó por la clasificación de Costa (1988), ya que se considera que es el que mejor explica la dinámica en la región. Los flujos de ladera y de quebrada se clasifican según su concentración volumétrica de sedimentos $C_s$ [-]. Costa (1988) define tres regímenes con umbrales basados en el comportamiento reológico de la mezcla:

| Tipo de flujo                                         | $C_s$ [vol.]  | Comportamiento reológico                          |
| ----------------------------------------------------- | ------------- | ------------------------------------------------- |
| **Flujo de agua** (_streamflow_)                      | $< 0.20$      | Newtoniano — transporte de fondo y suspensión     |
| **Flujo hiperconcentrado** (_hyperconcentrated flow_) | $0.20 - 0.47$ | Viscoplástico incipiente — yield strength medible |
| **Flujo de detritos** (_debris flow_)                 | $0.47 - 0.77$ | Visco-plástico — colisiones entre granos dominan  |
| **Deslizamiento** (_landslide_)                       | $> 0.77$      | Movimiento de masa — comportamiento sólido        |

> Los límites entre clases pueden solaparse, especialmente en el rango $0.20 < C_s < 0.47$ donde la transición es progresiva y depende de la granulometría y la densidad del fluido intersticial (Costa, 1988).

En SynxFlow, $C_s$ es calculada en cada celda y en cada paso de tiempo como variable de estado. La salida `C_{t}.asc` permite clasificar espacialmente el tipo de flujo a lo largo del evento.

## Comparación de modelos de flujo de detritos

Los modelos numéricos de flujo de detritos difieren principalmente en su formulación reológica del estrés basal $\tau$. La tabla siguiente compara los modelos más utilizados en la práctica profesional e investigativa:

| Modelo | Reología | Ecuación de estrés basal | Parámetros clave | Método numérico | Solver | Referencia |
| ------ | -------- | ------------------------ | ---------------- | --------------- | ------ | ---------- |
| **SynxFlow** | Manning + Mohr-Coulomb | $\tau = \rho g n^2 h^{-1/3}(u^2+v^2) + (\rho_s-\rho_f)\,g\,h\,C\,\tan\phi_d$ | $n$, $\phi_d$, $C$ | Vol. finito (Godunov) | HLLC Riemann — GPU (CUDA) | Xia et al. (2023), *Eng. Geol.* 326:107310 |
| **Flo-2D** | Cuadrática (O'Brien) | $\tau = \tau_y + \eta \dfrac{du}{dz} + \rho_m K \left(\dfrac{du}{dz}\right)^2$ | $\tau_y$, $\eta$, $K$, $n$ | Dif. finitas | Explícito (upwind) | O'Brien et al. (1993) |
| **RAMMS::DEBRIS FLOW** | Voellmy | $\tau = \mu\rho_m gh\cos\theta + \dfrac{\rho_m gu^2}{\xi}$ | $\mu$, $\xi$ [m/s²] | Vol. finito | Godunov-type | Christen et al. (2010) |
| **RiverFlow2D** | Múltiple: Manning, Coulomb, Bingham, cuadrática | 8 formulaciones (Tabla 1); fricción interna y basal desde agua limpia a mezcla hiperconcentrada | $n$, $\phi_b$, $\tau_y$, $\eta$, $K$ | Vol. finito | Roe Riemann aumentado — GPU | Murillo & García-Navarro (2012) |
| **EDDA 2.0** | Dos fases: Coulomb (sólido) + Manning (fluido) | $\tau = (\rho_s-\rho_f)gh\tan\phi_{bed} + \rho_f gn^2(u^2+v^2)h^{-1/3}$ | $n$, $\phi_{bed}$, $c_0$, $\phi_{bin}$, $\lambda$ (presión de poros) | Dif. finitas | MacCormack-TVD | Shen et al. (2018) |
| **DAN3D / DAN-W** | Variable (Coulomb, Voellmy, Bingham) | Seleccionable por evento | $\phi_b$, $\mu$, $\xi$, $\tau_y$ | Lagrangiano (SPH-type) | Lagrangiano continuo | McDougall & Hungr (2004) |
| **TITAN2D** | Coulomb (mezcla) | $\tau = (p - p_f)\tan\phi_b$ | $\phi_b$, $\phi_{int}$ | Vol. finito | Godunov-type | Pitman et al. (2003) |
| **D-Claw (USGS)** | Coulomb + presión de poros | $\tau = (\sigma - p_f)\tan\phi$ | $\phi$, $p_f$, dilatancia $\psi$ | Vol. finito | Godunov-type (GeoClaw) | George & Iverson (2014) |

### Notas sobre la selección de modelo

- **SynxFlow** y **RiverFlow2D** son los únicos modelos GPU-nativos de la tabla, lo que permite simular dominios de millones de celdas con tiempos de cómputo reducidos. RiverFlow2D emplea el solucionador de Riemann de Roe aumentado con 8 formulaciones reológicas (Murillo & García-Navarro, 2012, *J. Comput. Phys.* 231:1963–2001); su principal limitación para uso operacional es que es software comercial.
- **EDDA 2.0** acopla dos mecanismos de iniciación geotécnica (falla de ladera por saturación del suelo + escorrentía superficial) con la propagación del flujo. A diferencia de SynxFlow —que genera sedimento por erosión hidráulica del lecho a medida que el flujo supera el esfuerzo cortante crítico—, EDDA 2.0 incluye criterios explícitos de rotura geotécnica (presión de poros, cohesión, ángulo de fricción interna) como mecanismo de inicio de la masa (Shen et al., 2018, *Geosci. Model Dev.* 11:2841–2856).
- **Flo-2D** es ampliamente usado en estudios de riesgo por su interfaz comercial y reología cuadrática; no modela erosión dinámica del lecho.
- **RAMMS** es el estándar de facto para retroanálisis en Alpes y Andes; su fortaleza es la parametrización de Voellmy con solo dos parámetros ($\mu$, $\xi$).
- **DAN3D** es Lagrangiano (sin malla fija), útil para largos recorridos (_long-runout_); permite cambiar de reología a lo largo del trayecto.
- **D-Claw** es físicamente el más completo al resolver la presión de poros como variable de estado, pero requiere datos geotécnicos detallados.

### Justificación del uso de SynxFlow en el SAT del Rímac

SynxFlow fue seleccionado como motor de simulación del SAT por las siguientes razones:

- **Código abierto** (licencia GPL-3.0): permite modificación, integración y distribución sin restricciones comerciales. RiverFlow2D, con capacidades GPU comparables, es software de pago.
- **Integración Python/HPC**: la interfaz nativa en Python facilita la automatización del ciclo lluvia → simulación → postproceso en el entorno de clúster de SENAMHI.
- **Flujo completo en un solver**: integra lluvia → infiltración (Green-Ampt) → escorrentía → erosión hidráulica del lecho → propagación de la mezcla agua-sedimento, todo en un único kernel GPU sin transferencia de datos entre módulos.
- **Escalabilidad**: con 1 GPU H100, el dominio de 8.2 millones de celdas activas a 5 m de resolución se simula en 20 min–7 h según la intensidad del evento, compatible con los tiempos operacionales del SAT.

## Descripción general

El módulo de flujo de detritos de SynxFlow resuelve las ecuaciones de aguas poco profundas (SWE) extendidas para mezclas agua-sólido, incluyendo términos de fricción de Coulomb, erosión/deposición dinámica e infiltración. El solucionador está implementado en CUDA C++ y opera completamente en GPU, lo que permite simular dominios de millones de celdas en tiempos de cómputo del orden de horas.

A diferencia del módulo de inundación (_flood_), el módulo de detritos (_debris_) incorpora:

- Concentración volumétrica de sedimentos **C** como variable de estado adicional.
- Fricción de Coulomb dependiente de la concentración de sedimentos y el contraste de densidad sólido-fluido.
- Intercambio de masa entre el flujo y el lecho (erosión y deposición).
- Infiltración Green-Ampt acoplada directamente al balance de volumen.

## Ecuaciones de gobierno

El módulo resuelve las ecuaciones de conservación de masa y momento para una mezcla agua-sólido:

### Conservación de volumen de la mezcla

$$\frac{\partial h}{\partial t} + \frac{\partial (hu)}{\partial x} + \frac{\partial (hv)}{\partial y} = S_v - f$$

Donde $h$ es la profundidad de la mezcla [m], $u$ y $v$ son las componentes de velocidad [m/s], $S_v$ es el término fuente volumétrico (lluvia, afluentes) y $f$ es la tasa de infiltración Green-Ampt [m/s].

### Conservación de momento

$$\frac{\partial (hu)}{\partial t} + \frac{\partial}{\partial x}\left(hu^2 + \frac{g h^2}{2}\right) + \frac{\partial (huv)}{\partial y} = -gh\frac{\partial z}{\partial x} - \frac{\tau_x}{\rho_m}$$

$$\frac{\partial (hv)}{\partial t} + \frac{\partial (huv)}{\partial x} + \frac{\partial}{\partial y}\left(hv^2 + \frac{g h^2}{2}\right) = -gh\frac{\partial z}{\partial y} - \frac{\tau_y}{\rho_m}$$

Donde $z$ es la elevación del lecho [m], $g$ es la aceleración gravitacional [m/s²], $\rho_m$ es la densidad de la mezcla [kg/m³] y $\boldsymbol{\tau}$ es el estrés basal total.

### Conservación de sedimentos

$$\frac{\partial (hC)}{\partial t} + \frac{\partial (huC)}{\partial x} + \frac{\partial (hvC)}{\partial y} = E - D$$

Donde $C$ es la concentración volumétrica de sedimentos [-], $E$ es la tasa de erosión y $D$ es la tasa de deposición [m/s].

### Densidad de la mezcla

$$\rho_m = \rho_w (1 - C) + \rho_s C$$

Con $\rho_w = 1000$ kg/m³ (agua) y $\rho_s \approx 2650$ kg/m³ (sólidos).

## Modelo de fricción: Manning + Coulomb

El estrés basal total en el módulo de detritos combina una componente turbulenta tipo Manning (régimen fluido) y una componente granular tipo Coulomb (régimen denso), esta última escalada por la concentración de sedimentos $C$:

$$\tau = \rho \, g \, n^2 \, h^{-1/3}\left(u^2+v^2\right) + \left(\rho_s - \rho_f\right) g \, h \, C \, \tan\phi_d$$

Donde:

| Símbolo            | Parámetro SynxFlow       | Descripción                                                       | Valor típico             |
| ------------------ | ------------------------ | ------------------------------------------------------------------ | ------------------------- |
| $n$                | `manning`                | Coeficiente de Manning (componente fluida)                         | según cobertura ESA       |
| $\phi_d$           | `dynamic_friction_coeff`* | Ángulo de fricción dinámica del material granular ($\tan\phi_d$)  | equivalente a $\mu_d=0.44$ |
| $C$                | — (variable de estado)  | Concentración volumétrica de sedimentos                            | inicia en 0                |
| $\rho_s$, $\rho_f$ | —                        | Densidad del sólido y del fluido                                   | 2650, 1000 kg/m³           |

\* **Corrección respecto a una versión anterior de este documento:** se describía un segundo coeficiente independiente, `static_friction_coeff` ($\mu_s$), como término adicional del estrés basal con un factor $\cos\theta$ de pendiente. La ecuación publicada en Xia et al. (2023) usa un único coeficiente de fricción, $\tan\phi_d$, en el término granular de la ecuación de momento. `static_friction_coeff` sí existe como parámetro de la API de SynxFlow (confirmado en el binario `debris.so`), pero su rol parece corresponder al criterio de fluencia/inicio del movimiento (umbral estático de Mohr-Coulomb que determina si la celda fluye o permanece estática), no a un segundo término dentro de $\tau$. **Pendiente verificar contra el texto completo del paper** antes de calibrar ambos coeficientes como si fueran independientes en la ecuación de momento.

El valor de referencia ($\phi_d$ equivalente a $\mu_d=\tan\phi_d=0.44$, es decir $\phi_d\approx23.7°$) fue determinado por Xia et al. (2023) mediante calibración con el experimento de canal de Takahashi et al. (1992), y constituye el punto de partida para la calibración en la cuenca del Rímac.

### Valores de retroanálisis en cuencas andinas áridas y semiáridas

Los siguientes valores de fricción tipo Coulomb han sido determinados mediante retroanálisis en zonas andinas de condiciones similares a la cuenca del Rímac, y sirven como rango orientativo para la calibración:

| Sitio | País | Condición | Modelo | $\mu$ | Referencia | Página |
| ----- | ---- | --------- | ------ | ----- | ---------- | ------ |
| Takahashi flume | Lab. | Experimento canal | SynxFlow | $\mu_s=0.62$, $\mu_d=0.44$ | Xia et al. (2023), *Eng. Geol.* 326:107310 | §3 (p. 6) |
| Quebrada Sahuanay, Abancay | Perú | Fase seca | RAMMS | $\mu=0.20$ | Rodríguez-Morata et al. (2019), *Geomorphology* 342:127–139 | p. 133 |
| Quebrada Sahuanay, Abancay | Perú | Fase húmeda | RAMMS | $\mu=0.10$ | Rodríguez-Morata et al. (2019), *Geomorphology* 342:127–139 | p. 133 |
| Lago 513, Carhuaz (GLOF granular) | Perú | Flujo granular | RAMMS | $\mu=0.16$ | Schneider et al. (2014), *Adv. Geosci.* 35:145–155 | p. 149 |
| Lago 513, Carhuaz (GLOF viscoso) | Perú | Flujo viscoso | RAMMS | $\mu=0.08$ | Schneider et al. (2014), *Adv. Geosci.* 35:145–155 | p. 149 |
| Lago 513, Carhuaz (hiperconc.) | Perú | Hiperconcentrado | RAMMS | $\mu=0.04$ | Schneider et al. (2014), *Adv. Geosci.* 35:145–155 | p. 149 |
| Manflas, Andes áridos | Chile | GLOF — largo recorrido | RAMMS | $\mu \leq 0.001$ | Iribarren Anacona et al. (2018), *Nat. Hazards* 94:93–119 | p. 104 |

> **Nota:** Los valores de Sahuanay, Carhuaz y Manflas fueron extraídos de la tabla de revisión de Mikoš & Bezak (2021), pp. 2, 5 y 6 respectivamente, donde se consolidan más de 30 retroanálisis con RAMMS en distintos ambientes montañosos. Los valores de $\mu$ de RAMMS (fricción Voellmy) no son directamente equivalentes a los de SynxFlow, pero son orientativos del rango esperable para estos materiales. En SynxFlow, $\mu_d=\tan\phi_d=0.44$ es el que entra directamente en $\tau$; $\mu_s=0.62$ corresponde al umbral estático de fluencia (ver nota en la sección anterior).

## Erosión y deposición

### Profundidad erosionable

El lecho disponible para erosión se limita mediante el parámetro `erodible_depth` [m], que representa el espesor máximo de material que puede ser removido. Para fines de este manual, los valores de profundidad erosionable se establecieron a partir del producto ESA. Este valor varía por clase de cobertura de suelo:

| Clase ESA WorldCover | Cobertura            | `erodible_depth` [m] |
| -------------------- | -------------------- | -------------------- |
| 10 — Tree cover      | Bosque               | 0.5                  |
| 20 — Shrubland       | Matorral             | 0.3                  |
| 30 — Grassland       | Pastizal             | 0.3                  |
| 40 — Cropland        | Cultivos             | 0.4                  |
| 50 — Built-up        | Zona urbana          | 0.0                  |
| 60 — Bare/sparse     | Suelo desnudo / roca | 0.8                  |
| 80 — Water bodies    | Cuerpos de agua      | 0.0                  |
| 90 — Wetland         | Humedal              | 0.1                  |

La clase 60 (suelo desnudo/roca, dominante en la cuenca alta andina) tiene el mayor valor por la disponibilidad de material coluvial no consolidado. Este valor es el que se analiza su sensibilidad y se calibra.

### Variable de estado `erodible_depth`

`erodible_depth` es una variable que **cambia durante la simulación**: disminuye a medida que el material es erosionado y aumenta por deposición. Se escribe como salida periódica y como archivo de respaldo (_backup_), lo que permite:

- Inspeccionar la distribución espacial de la erosión acumulada.
- Retomar una simulación desde un estado intermedio con el lecho correcto.

## Concentración inicial

La concentración volumétrica de sedimentos se inicializa como cero (`C = 0`) al inicio de la simulación, representando agua limpia. El modelo genera sedimentos progresivamente conforme el flujo erosiona el lecho.

La condición de contorno aguas arriba puede especificar una concentración de entrada (`C_in`), útil para representar caudales con carga sedimentaria proveniente de subcuencas no modeladas explícitamente.

## Variables de salida del módulo de detritos

| Archivo                    | Variable | Unidades | Descripción                        |
| -------------------------- | -------- | -------- | ---------------------------------- |
| `h_{t}.asc`                | $h$      | m        | Profundidad de la mezcla           |
| `hUx_{t}.asc`              | $hu$     | m²/s     | Descarga unitaria en X             |
| `hUy_{t}.asc`              | $hv$     | m²/s     | Descarga unitaria en Y             |
| `cumulative_depth_{t}.asc` | $F$      | m        | Infiltración acumulada             |
| `erodible_depth_{t}.asc`   | $d_e$    | m        | Profundidad erosionable remanente  |
| `h_gauges.dat`             | $h(t)$   | m        | Serie temporal en gauges virtuales |

La velocidad de la mezcla se calcula en postprocesamiento como:

$$v = \frac{\sqrt{(hu)^2 + (hv)^2}}{h}$$

## Configuración en Python

```python
from synxflow import IO, debris

# Parámetros reológicos (uniformes en el dominio)
static_friction  = np.full(DEM.array.shape, 0.62)  # umbral de fluencia (inicio del movimiento); no aparece en tau
dynamic_friction = np.full(DEM.array.shape, 0.44)  # tan(phi_d): término de Coulomb en tau
concentration    = np.zeros(DEM.array.shape)   # agua limpia inicial

# Profundidad erosionable por clase ESA
ero_depth = {10: 0.5, 20: 0.3, 30: 0.3, 40: 0.4,
             50: 0.0, 60: 0.8, 80: 0.0, 90: 0.1}
ero_array = make_grid(ero_depth)

# Asignación de parámetros al modelo
case_input.add_user_defined_parameter('C',                      concentration)
case_input.add_user_defined_parameter('erodible_depth',         ero_array)
case_input.add_user_defined_parameter('static_friction_coeff',  static_friction)
case_input.add_user_defined_parameter('dynamic_friction_coeff', dynamic_friction)

# Ejecución en GPU (1 GPU)
debris.run(case_folder)

# Ejecución en múltiples GPU
debris.run_mgpus(case_folder)
```

## Consideraciones para cuencas andinas

En cuencas de montaña tipo quebrada (pendientes >5%, material coluvial), se recomienda:

- **`static_friction_coeff`**: calibrar en el rango 0.55–0.70 según granulometría del material.
- **`dynamic_friction_coeff`**: típicamente 0.65–0.80 del valor estático.
- **`erodible_depth`**: verificar con levantamientos geomorfológicos post-evento.
- La concentración de salida `C` puede compararse con muestreos de carga sedimentaria en la estación hidrométrica.
