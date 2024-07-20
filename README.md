# JUPITER Benchmark Suite: nekRS

[![DOI](https://zenodo.org/badge/831414161.svg)](https://zenodo.org/badge/latestdoi/831414161) [![Static Badge](https://img.shields.io/badge/DOI%20(Suite)-10.5281%2Fzenodo.12737073-blue)](https://zenodo.org/badge/latestdoi/764615316)

This benchmark is part of the [JUPITER Benchmark Suite](https://github.com/FZJ-JSC/jubench). See the repository of the suite for some general remarks.

This repository contains the nekRS benchmark. [`DESCRIPTION.md`](DESCRIPTION.md) contains details for compilation, execution, and evaluation.

[NekRS](https://github.com/Nek5000/nekRS) is an open-source GPU-accelerated Navier Stokes solver based on the spectral element method.  
The provided case is a mesoscale convection case. It is based on the RBC (Rayleigh-Bénard convection) example of nekRS and calculates the turbulence induced by a temperature gradient. Depending on the grid resolution, this allows simulations of high Rayleigh numbers while using low Prandtl numbers. To keep the case similar for all cases, and the low resolution for the 1 node cases, the Rayleigh number will be 100000 and the Prandtl number 0.7. The simulation domain is a "sheet", therefore much greater extent in X and Y direction than Z direction. These scaling tests run just a short time, therefore still in the initialization of the fluid motion, doing 600 time steps.

The source code of nekRS is included in the `./src/` subdirectory as a submodule from the upstream nekRS repository at [github.com/Nek5000/nekRS](https://github.com/Nek5000/nekRS).

## Quickstart

### Execution

Overview over the provided cases, the baseline case is the default option.

| **Name**         | **JUBE flag** | **Node counts**         | **GPU Memory Utilization on A100** |
|------------------|---------------|-------------------------|----------------------------|
| baseline         | -             |             8           | 100%                       |
| scaling          | scaling       | 1, 2, 4, 6, 8           | 100%   (on one node)       |
| large_scaling    | large_scaling | 64, 128, 256, 384       | 100%   (on 64 nodes)       |
| high_large       | high_large    |                642      | 100%                       |
| high_medium      | high_medium   |                642      | 75%                        |
| high_small       | high_small    |                642      | 50%                        |


### Running

The JUBE step `execute` will submit the job to the batch system, by using the batch submission script template (via `platform.xml`) with information specified in the top of the script relating to the number of nodes and tasks per node.
Via dependencies, the JUBE step `execute` calls the JUBE steps that compile nekRS and genbox automatically.

To submit a self-contained benchmark run to the batch system, call `jube run benchmark/jube/default.yaml`.
JUBE will generate the necessary configuration and files, and submit the benchmark to the batch engine.

The following parameters of the JUBE script might need to be adapted:
- `taskspernode` and `gres`: Should be equal to the number of GPUs
- `threadspertask`: Divide your threads equally onto all tasks
- `queue` and `account`: SLURM queue and account to use
- `modules`: To be sourced before building and running

Additional JUBE flags can be used to differentiate between the runs:
`jube run benchmark/jube/default.yaml --tag=scaling`


### Results
Once all runs are completed, the results can be generated with `jube result -a src --style csv`. JUBE will also write the generated table into `src/000000/result/result.dat` into csv format.
