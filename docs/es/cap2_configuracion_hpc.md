# Configuración de SynxFlow en HPC

## Descripción del entorno

SynxFlow requiere un entorno con soporte CUDA para la ejecución en GPU. El modelo es compatible con cualquier clúster HPC que disponga de GPUs NVIDIA con capacidad de cómputo sm_60 o superior y CUDA 11.0+.

**Especificaciones mínimas recomendadas:**

| Componente          | Mínimo                | Recomendado               |
| ------------------- | --------------------- | ------------------------- |
| GPU                 | NVIDIA con CUDA sm_60 | NVIDIA A100 / H100        |
| Memoria GPU         | 16 GB                 | 40–80 GB                  |
| CPUs                | 8 cores               | 32+ cores                 |
| Memoria RAM         | 32 GB                 | 64–100 GB                 |
| CUDA                | 11.0                  | 12.x                      |
| Sistema de archivos | local                 | scratch de alta velocidad |

## Instalación simple (paquete oficial)

La forma más sencilla de instalar SynxFlow es mediante `pip` desde el repositorio oficial. Se recomienda usar un entorno Conda dedicado con Python 3.9 o superior.

```bash
# Crear entorno Conda
conda create -n synxflow_env python=3.9 -y
conda activate synxflow_env

# Instalar dependencias científicas
pip install numpy pandas matplotlib rasterio geopandas xarray rioxarray

# Instalar SynxFlow (versión oficial desde PyPI o GitHub)
pip install synxflow
```

> **Nota:** La instalación vía `pip install synxflow` descarga el paquete precompilado para las arquitecturas GPU más comunes. Si su GPU no está soportada, consulte la sección de instalación en modo desarrollo en la documentación oficial del proyecto en GitHub.

### Verificar la instalación

```python
import synxflow
from synxflow import IO, debris, flood
print(synxflow.__version__)
```

### Registro del kernel Jupyter (opcional)

```bash
conda activate synxflow_env
pip install jupyter ipykernel
python -m ipykernel install --user --name synxflow_env \
       --display-name "Python (synxflow_env)"
```

## Configuración del job SLURM

### Job estándar (1 GPU, 1 simulación)

```bash
#!/bin/bash
#SBATCH --job-name=simulacion
#SBATCH --partition=gpu
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --time=1-00:00:00        # tiempo máximo: 1 día
#SBATCH --mem=64G
#SBATCH --output=simulacion_%j.out
#SBATCH --error=simulacion_%j.err

# Cargar módulos del sistema (ajustar según el clúster)
module purge
module load cuda

# Activar entorno Conda
source $HOME/miniconda3/etc/profile.d/conda.sh
conda activate synxflow_env

# Verificar GPU disponible
nvidia-smi

# Ejecutar simulación
python -u /ruta/al/script_simulacion.py
```

### Array job (múltiples versiones, N simultáneas)

Para análisis de sensibilidad o calibración con múltiples juegos de parámetros, se usa un array job con límite de concurrencia `%N`:

```bash
#!/bin/bash
#SBATCH --job-name=sensibilidad
#SBATCH --partition=gpu
#SBATCH --array=1-5%3           # versiones 1-5, máximo 3 en paralelo
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --time=1-00:00:00
#SBATCH --mem=64G
#SBATCH --output=sens_v%a_%j.out
#SBATCH --error=sens_v%a_%j.err

source $HOME/miniconda3/etc/profile.d/conda.sh
conda activate synxflow_env

python -u /ruta/al/script_parametrizado.py
# El script lee SLURM_ARRAY_TASK_ID para determinar la versión
```

El script Python lee la versión del entorno:

```python
import os, sys
task_id = int(os.environ.get('SLURM_ARRAY_TASK_ID',
              sys.argv[1] if len(sys.argv) > 1 else '1'))
```

### Selección del número de GPUs simultáneas

El clúster dispone de **3 nodos GPU**, cada uno con **2× H100** (6 GPUs en total). Cada job solicita 1 GPU y ~91 CPUs; dado que cada nodo tiene 192 CPUs, pueden coexistir hasta 2 jobs por nodo.

| Jobs simultáneos | GPUs usadas | GPUs libres (6 total) | Uso recomendado                   |
| ---------------- | ----------- | --------------------- | --------------------------------- |
| `%1`             | 1           | 5                     | Producción (sin interferencia)    |
| `%3`             | 3           | 3                     | Calibración/sensibilidad          |
| `%5`             | 5           | 1                     | Uso intensivo (dejar 1 GPU libre) |
| `%6`             | 6           | 0                     | Uso exclusivo del clúster GPU     |

\* Para la etapa de analisis de sensibilidad, se sugiere usar mas de una GPU pero sin monopolizar las GPU en el HPC.

## Tiempos de cómputo de referencia

Los tiempos siguientes corresponden a la cuenca del río Rímac (dominio de 8.2 millones de celdas activas, resolución 5 m, 1 GPU H100):

| Duración sim.      | Condición                 | Tiempo de pared aprox. |
| ------------------ | ------------------------- | ---------------------- |
| 3 horas            | Evento seco               | ~16 minutos            |
| 24 horas           | Evento húmedo (Yaku 2023) | ~2.5 horas             |
| 192 horas (8 días) | Evento completo           | ~20 horas              |

> El tiempo de cómputo escala aproximadamente de forma lineal con la duración simulada (~6 min/hora-simulada). En condiciones de lluvia intensa (mayor número de celdas activas y paso de tiempo reducido), el tiempo puede aumentar hasta un factor 1.5×.

## Gestión de almacenamiento

Las simulaciones generan volúmenes grandes de datos. Se recomienda escribir los resultados en el sistema de archivos scratch (alta velocidad, sin cuota):

| Configuración de salida | Archivos por variable | Tamaño aprox. por variable |
| ----------------------- | --------------------- | -------------------------- |
| Cada 15 min, 24h        | 96 rasters            | ~18 GB                     |
| Cada 1h, 24h            | 24 rasters            | ~4.5 GB                    |
| Cada 1h, 192h           | 192 rasters           | ~36 GB                     |

**Variables de salida del debris solver:** `h`, `hUx`, `hUy`, `cumulative_depth`, `erodible_depth` → multiplicar por 5 el tamaño estimado por variable.

```python
# Escribir salidas en scratch (recomendado)
SCRATCH = '/scratch/workdir/<usuario>/simulaciones'
case_folder = os.path.join(SCRATCH, 'nombre_corrida')

# Configurar intervalos: [t_ini, t_fin, dt_output, dt_backup]
case_input.set_runtime([
    0,
    3600 * 192,   # 192 horas
    3600 * 1,     # salida cada 1h
    3600 * 24,    # backup cada 24h
])
```

## Monitoreo de simulaciones

### Estado de los jobs

```bash
# Ver todos los jobs del usuario
squeue -u <usuario> --format="%.10i %.14j %.8T %.10M %R"

# Ver detalle de un job específico
scontrol show job <jobid>
```

### Progreso de la simulación

```bash
# Último timestep generado (h en el canal)
ls /scratch/.../output/h_*.asc | sort -t_ -k2 -n | tail -3

# Log del job en tiempo real
tail -f simulacion_<jobid>.out
```

### Verificación del dominio

```python
import numpy as np

data = np.loadtxt('/scratch/.../output/h_900.asc', skiprows=6)
print(f"Celdas activas : {(data != -9999).sum():,}")
print(f"Celdas con h>0 : {(data > 0).sum():,}")
print(f"h_max          : {data[data != -9999].max():.3f} m")
```

## Errores comunes y soluciones

Esto es oro, así que considerenlo, otras soluciones a problemas pueden encontrarse en el repositorio oficial de GitHub del modelo:

| Error                                               | Causa                                            | Solución                                                                                                   |
| --------------------------------------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `GLIBCXX_3.4.31 not found` al cargar GDAL           | Conflicto entre GDAL de spack y plugins de conda | `export GDAL_DRIVER_PATH=""` antes de `gdal2tiles.py`                                                      |
| Job sin output después de varios minutos            | Error en la inicialización del modelo            | Revisar el `.err` del job; típicamente parámetro inválido o archivo de entrada ausente                     |
| Simulación con `dt` estancado en 0.005s             | Inestabilidad numérica                           | Verificar condiciones de contorno (h inicial, flujo de entrada); revisar si el DEM tiene pits sin rellenar |
| `ModuleNotFoundError: typing_extensions` en Jupyter | Dependencia faltante en el entorno               | `pip install typing_extensions` en el entorno activo                                                       |
