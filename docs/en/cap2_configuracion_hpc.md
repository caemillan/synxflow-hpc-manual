# Configuring SynxFlow on HPC

## Environment description

SynxFlow requires an environment with CUDA support for GPU execution. The model is compatible with any HPC cluster that has NVIDIA GPUs with compute capability sm_60 or higher and CUDA 11.0+.

**Minimum recommended specifications:**

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU | NVIDIA with CUDA sm_60 | NVIDIA A100 / H100 |
| GPU memory | 16 GB | 40–80 GB |
| CPUs | 8 cores | 32+ cores |
| RAM | 32 GB | 64–100 GB |
| CUDA | 11.0 | 12.x |
| File system | local | high-speed scratch |

## Simple installation (official package)

The easiest way to install SynxFlow is via `pip` from the official repository. It is recommended to use a dedicated Conda environment with Python 3.9 or higher.

```bash
# Create Conda environment
conda create -n synxflow_env python=3.9 -y
conda activate synxflow_env

# Install scientific dependencies
pip install numpy pandas matplotlib rasterio geopandas xarray rioxarray

# Install SynxFlow (official version from PyPI or GitHub)
pip install synxflow
```

> **Note:** Installing via `pip install synxflow` downloads the pre-compiled package for the most common GPU architectures. If your GPU is not supported, consult the development mode installation section in the official project documentation on GitHub.

### Verify the installation

```python
import synxflow
from synxflow import IO, debris, flood
print(synxflow.__version__)
```

### Jupyter kernel registration (optional)

```bash
conda activate synxflow_env
pip install jupyter ipykernel
python -m ipykernel install --user --name synxflow_env \
       --display-name "Python (synxflow_env)"
```

## SLURM job configuration

### Standard job (1 GPU, 1 simulation)

```bash
#!/bin/bash
#SBATCH --job-name=simulacion
#SBATCH --partition=gpu
#SBATCH --ntasks=1
#SBATCH --nodes=1
#SBATCH --gres=gpu:1
#SBATCH --time=1-00:00:00        # maximum time: 1 day
#SBATCH --mem=64G
#SBATCH --output=simulacion_%j.out
#SBATCH --error=simulacion_%j.err

# Load system modules (adjust to the cluster)
module purge
module load cuda

# Activate Conda environment
source $HOME/miniconda3/etc/profile.d/conda.sh
conda activate synxflow_env

# Verify available GPU
nvidia-smi

# Run simulation
python -u /ruta/al/script_simulacion.py
```

### Array job (multiple versions, N simultaneous)

For sensitivity analysis or calibration with multiple parameter sets, use an array job with a concurrency limit `%N`:

```bash
#!/bin/bash
#SBATCH --job-name=sensibilidad
#SBATCH --partition=gpu
#SBATCH --array=1-5%3           # versions 1-5, maximum 3 in parallel
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
# The script reads SLURM_ARRAY_TASK_ID to determine the version
```

The Python script reads the version from the environment:

```python
import os, sys
task_id = int(os.environ.get('SLURM_ARRAY_TASK_ID',
              sys.argv[1] if len(sys.argv) > 1 else '1'))
```

### Selecting the number of simultaneous GPUs

The cluster has **3 GPU nodes**, each with **2× H100** (6 GPUs in total). Each job requests 1 GPU and ~91 CPUs; since each node has 192 CPUs, up to 2 jobs can coexist per node.

| Simultaneous jobs | GPUs used | GPUs free (6 total) | Recommended use |
|------------------|-----------|---------------------|----------------|
| `%1` | 1 | 5 | Production (no interference) |
| `%3` | 3 | 3 | Calibration/sensitivity |
| `%5` | 5 | 1 | Intensive use (leave 1 GPU free) |
| `%6` | 6 | 0 | Exclusive cluster use |

\* For sensitivity analysis, it is recommended to use more than one GPU without monopolising all available GPUs on the cluster.

## Reference computation times

The following times correspond to the Rímac River watershed (domain of 8.2 million active cells, 5 m resolution, 1 H100 GPU):

| Sim. duration | Condition | Approximate wall time |
|--------------|-----------|----------------------|
| 3 hours | Dry event | ~16 minutes |
| 24 hours | Wet event (Yaku 2023) | ~2.5 hours |
| 192 hours (8 days) | Complete event | ~20 hours |

> Computation time scales approximately linearly with simulated duration (~6 min/simulated-hour). Under heavy rainfall conditions (more active cells and reduced time step), the time can increase by up to a factor of 1.5×.

## Storage management

Simulations generate large data volumes. It is recommended to write results to the scratch file system (high speed, no quota):

| Output configuration | Files per variable | Approximate size per variable |
|---------------------|-------------------|------------------------------|
| Every 15 min, 24h | 96 rasters | ~18 GB |
| Every 1h, 24h | 24 rasters | ~4.5 GB |
| Every 1h, 192h | 192 rasters | ~36 GB |

**Debris solver output variables:** `h`, `hUx`, `hUy`, `cumulative_depth`, `erodible_depth` → multiply estimated size per variable by 5.

```python
# Write outputs to scratch (recommended)
SCRATCH = '/scratch/workdir/<usuario>/simulaciones'
case_folder = os.path.join(SCRATCH, 'nombre_corrida')

# Configure intervals: [t_start, t_end, dt_output, dt_backup]
case_input.set_runtime([
    0,
    3600 * 192,   # 192 hours
    3600 * 1,     # output every 1h
    3600 * 24,    # backup every 24h
])
```

## Simulation monitoring

### Job status

```bash
# View all jobs for the user
squeue -u <usuario> --format="%.10i %.14j %.8T %.10M %R"

# View details of a specific job
scontrol show job <jobid>
```

### Simulation progress

```bash
# Last generated timestep (h in channel)
ls /scratch/.../output/h_*.asc | sort -t_ -k2 -n | tail -3

# Job log in real time
tail -f simulacion_<jobid>.out
```

### Domain verification

```python
import numpy as np

data = np.loadtxt('/scratch/.../output/h_900.asc', skiprows=6)
print(f"Active cells   : {(data != -9999).sum():,}")
print(f"Cells with h>0 : {(data > 0).sum():,}")
print(f"h_max          : {data[data != -9999].max():.3f} m")
```

## Common errors and solutions

The following are hard-won lessons — keep them in mind. Additional solutions may be found in the model's official GitHub repository.

| Error | Cause | Solution |
|-------|-------|---------|
| `GLIBCXX_3.4.31 not found` when loading GDAL | Conflict between spack GDAL and conda plugins | `export GDAL_DRIVER_PATH=""` before `gdal2tiles.py` |
| Job with no output after several minutes | Error during model initialisation | Check the job `.err` file; typically an invalid parameter or missing input file |
| Simulation with `dt` stuck at 0.005s | Numerical instability | Check boundary conditions (initial h, inflow); check whether the DEM has unfilled pits |
| `ModuleNotFoundError: typing_extensions` in Jupyter | Missing dependency in the environment | `pip install typing_extensions` in the active environment |
