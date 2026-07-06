# Introducción

Bienvenido al Manual de referencia para la simulación hidrodinámica de amenazas naturales en HPC con SynxFlow. El presente manual introduce los conceptos técnicos para la modelación hidrodinámica con SynxFlow para la implementación de los módulos de hidrología y flujo de detritos en el modelamiento a escala de quebrada y cuenca en un entorno de computación de alto rendimiento (HPC).

Este manual ha sido elaborado por **Carlos Millán-Arancibia** (Servicio Nacional de Meteorología e Hidrología del Perú — SENAMHI, Perú) y **Xilin Xia** (Universidad de Birmingham, Reino Unido) como parte del desarrollo del Sistema de Alerta Temprana (SAT) para la cuenca media-baja del río Rímac. Está dirigido a profesionales en hidrología, ingeniería hidráulica y ciencias de la tierra que requieran implementar simulaciones hidrodinámicas 2D con aceleración GPU en entornos de computación de alto rendimiento.

## Alcance

El manual cubre:

1. **Descripción del modelo SynxFlow** — fundamentos de las ecuaciones de aguas poco profundas (SWE), módulos disponibles y capacidades de simulación.
2. **Configuración del entorno HPC** — instalación, compilación y ejecución en clústeres con GPU NVIDIA.
3. **Módulo de flujo de detritos** — ecuaciones de gobierno, modelo reológico de Coulomb, erosión/deposición e infiltración acoplada.
4. **Módulo de infiltración Green-Ampt** — implementación en GPU, parámetros por clase de cobertura y consideraciones para cuencas andinas semi-áridas.

## Contexto: Sistema de Alerta Temprana para el río Rímac

La cuenca del río Rímac, con su desembocadura en la ciudad de Lima (población ~10 millones), es una de las cuencas con mayor riesgo hidrometeorológico del Perú durante la epoca de lluvias. El área de abarcada concentra la exposición a flujos de detritos generados en quebradas tributarias (e.g. Huascarán, Cusipata, Los Condores, etc) durante eventos de lluvia intensa asociados a fenómenos como El Niño Costero.

El evento Yaku (9–16 marzo 2023) provocó flujos de detritos y desbordamientos en múltiples quebradas, afectando severamente la infraestructura vial y residencial de Chosica. Las simulaciones documentadas en este manual corresponden al modelamiento de ese evento como caso de back-analysis (retroanálisis) para la calibración y validación del SAT.

### Componentes del SAT

```
FTP (WRF 1 km rainfall)
       │
       ▼
Preprocesamiento de lluvia
(deacumulación, reproyección, recorte a cuenca)
       │
       ▼
Simulación GPU — SynxFlow (debris solver + Green-Ampt)
DEM 5m · 8.2 millones de celdas · ~20 min a 7h según evento
       │
       ▼
Postprocesamiento
(h_max, v_max, nivel de amenaza HR = f(h,v)
       │
       ▼
Publicación de teselas (formato de visualización de alto rendimiento de raster)
       │
       ▼
Plataforma web (SILVIA CHIRILU v1p3)
```

### Dominio de simulación

| Parámetro          | Valor                                 |
| ------------------ | ------------------------------------- |
| Área de simulación | Bajo Rímac — Lima                     |
| DEM                | SPOT-6, 5 m de resolución, EPSG:32718 |
| Celdas totales     | ~17.5 millones                        |
| Celdas activas     | ~8.2 millones. (47%)                  |
| Extensión          | ~23.9 km (E-O) × 18.3 km (N-S)        |

## Estructura del manual

| Capítulo        | Contenido                                                                                |
| --------------- | ---------------------------------------------------------------------------------------- |
| **Cap. 1**      | Introducción: contexto SAT, dominio de simulación, estructura del manual                 |
| **Cap. 2**      | Descripción de SynxFlow: ecuaciones SWE, módulos, metodología, casos de estudio          |
| **Cap. 3**      | Configuración del entorno HPC: instalación, SLURM, tiempos de referencia                 |
| **Cap. 4**      | Módulo de flujo de detritos: ecuaciones, fricción de Coulomb, erosión, parámetros        |
| **Cap. 5**      | Módulo de infiltración Green-Ampt: implementación GPU, parámetros Rawls, cuencas andinas |
| **Referencias** | Bibliografía completa citada en el manual                                                |
