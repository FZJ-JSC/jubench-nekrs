# nekRS

[nekRS](https://github.com/Nek5000/NekRS) is an Open-Source, GPU-accelerated Navier Stokes solver based on the spectral element method.

The provided benchmark case is a mesoscale convection simulation. It is based on the RBC (Rayleigh-Bénard convection) example of nekRS and simulates turbulence induced by a temperature gradient. Depending on the grid resolution, this allows simulations of high Rayleigh numbers while using low Prandtl numbers. To keep the case similar for all cases, and the low resolution for the 1 node cases, the Rayleigh number is selected to be 100000 and the Prandtl number 0.7. The simulation domain is a "sheet", therefore much greater extent in X and Y direction than Z direction. These scaling tests run just a short time, therefore still in the unsteady phase of the fluid motion, doing 600 time steps. 

## Source

Archive Name: `nekrs-bench.tar.gz`

The file holds instructions to run the benchmark, according JUBE scripts, and the configuration files for the nekRS cases.  
The nekRS source code is distributed in the `src/nekRS` directory. The version of nekRS included in the archive is [git commit hash `f7ecfa9d`](https://github.com/Nek5000/nekRS/tree/f7ecfa9db96e8476f249bba0f0eb4f8af2e612ac).

## Building

nekRS requirements:

- C++17 compatible compiler
- C11 compatible compiler 
- GNU Fortran
- MPI-3.1 or later
- CMake version 3.17 (AMGx requires >=3.18) or later

### OCCA

nekRS relies on [OCCA](https://github.com/libocca/occa) as API for multiple GPU-based programming models. OCCA is included in the source tree of nekRS and shipped within the benchmark and also configured in the CMake steps outline below.  
Using OCCA, nekRS builds hardware optimized kernels before each execution (by default) that are stored locally in a `.cache` folder. The case (incl. JUBE) should consider this automatically. However, as a consequence of other manual errors, removing this cache might sometimes be necessary.

### Compiler Flags

nekRS comes with pre-set compiler flags (incl. OCCA) that try to optimize the code performance. These are a compromise of runtime performance and numerical accuracy. It is not allowed to reduce the time to solution/scaling by decreasing the numerical accuracy by, for example, using particular approximate divisions or similar numerical options.

### JUBE

A JUBE script is provided (see below) which will automatically build nekRS and the required genbox tool when executing.

### Manual

nekRS will default to installing in the home directory. To change the install location, modify the `nrsconfig` file in the root directory of nekRS and set `NEKRS_INSTALL_DIR` (line 5) to change the install location.

Start the build process:

```bash
./nrsconfig
cmake --build ./build --target install -j8
```

#### Setting up Environment

After building and installing nekRS, you might want to add it to the environment (e.g. in your `~/.bashrc`), using the same value as for `NEKRS_INSTALL_DIR`.

```bash
export NEKRS_HOME=<NEKRS_INSTALL_DIR>
export PATH=$NEKRS_HOME/bin:$PATH
```

Otherwise you need to specify it for your job script.

#### Building Genbox Tool

We provide a modified version of the genbox tool, sourced from the nek5000 source code, which is responsible for generating the binary input file of the geometry for nekRS.
Our provided version was tweaked to allow the larger problem sizes we require.

The source can be found in `src/genbox` and can be built with the included Makefile.

```bash
cd src/genbox
make
```

#### Input Data

To run the benchmark cases, the grid used by the simulation needs to be built.

1. Execute genbox with `./genbox`.
2. genbox will ask for the input file name, `src/grids/` contains the grid specifications files for all runs with names along the configurations explained in the table below.
3. The generated file, `box.re2`, will have to be renamed to `rbc.re2`.

Depending on the execution variant of the benchmark, different grid specifications input files need to be taken. See below.

## Execution

### CPUs, Cores, Threads

nekRS performs all calculations on the GPUs with one MPI rank per GPU. The CPUs are of significance mostly only for I/O. The default is one thread per task. The number of OpenMP threads can be smaller than the number of cores. The number of threads can be adapted to match the best performance of the system.  
The shipped example Slurm script `src/submit.job` contains a well-performing configuration for JUWELS Booster, containing nodes with 2 CPU sockets (with one AMD EPYC 7402 processor with 24 cores per socket) and 4 NVIDIA A100 GPUs.

### Structure

Each run should be executed in its own folder which we call _case directory_. If JUBE is used, a folder structure will be created automatically. To run manually, you need to create the case directory and copy the previously made `rbc.re2` file as well as the other input files located in `src/mesoscale` to the case directory. After placing all necessary input files into the case directory, the files `rbc.oudf`, `rbc.par`, `rbc.re2`, `rbc.udf`, and `rbc.usr` should be located there.

The nekRS executable is to be launched from this case directory.

### Benchmark Variants

The nekRS benchmark is to be executed in two variants.

1. **TCO Baseline**: This variant is the benchmark for the TCO evaluation. The benchmark takes about 250 s.
2. **High-Scaling**: This variant uses a larger simulation box to explore scalability between a 50 PFLOP/s sub-partition of JUWELS Booster and a 1000 PFLOP/s sub-partition of the Exascale system (with 20x the performance). Three sub-variants are prepared, each utilizing different amounts of A100 GPU memory: _large_ (100% memory / 40 GB, minus a margin), _medium_ (75% / 30 GB, minus margin), and _small_ (50% / 20 GB, minus margin).

The variants and sub-variants are each using different grid specification files with fitting inputs. See the overview table for a list.

### Overview Table

The following table gives an overview of the provided cases, the baseline case is the default case.

|    **Name**   |  **JUBE Tag**   | **Reference Node Counts** | **GPU Memory** |   **Grid Input**  |
|---------------|-----------------|---------------------------|----------------| ----------------- |
| baseline      | -               | 8                         | 100%           | `baseline.box`    |
| high_large    | `high_large`    | 642                       | 100%           | `high_large.box`  |
| high_medium   | `high_medium`   | 642                       | 75%            | `high_medium.box` |
| high_small    | `high_small`    | 642                       | 50%            | `high_small.box`  |
| exa_large     |                 | 12840                     | expected 100%  | `exa_large.box`   |
| exa_medium    |                 | 12840                     | expected 75%   | `exa_medium.box`  |
| exa_small     |                 | 12840                     | expected 50%   | `exa_small.box`   |

The _GPU Memory_ column refers to GPU memory usage of an NVIDIA A100-40GB GPU.

The cases `exa_large`, `exa_medium` and `exa_small` are the cases to run on an exascale system. The three `exa_*` cases have the same ratios of `required memory / performance` as the respective `high_*` cases.

In addition, further tags exist in the JUBE script for further exploration (`scaling`, `large_scaling`).

### JUBE

The JUBE step `execute` will submit the job to the batch system, by using the batch submission script template (via `platform.xml`) with information specified in the top of the script relating to the number of nodes and tasks per node.
Via dependencies, the JUBE step `execute` calls the JUBE steps that compile nekRS and genbox automatically.

To submit a self-contained benchmark run to the batch system, call `jube run benchmark/jube/default.yml`.
JUBE will generate the necessary configuration and files, and submit the benchmark to the batch engine.

The following parameters of the JUBE script might need to be adapted:
- `taskspernode` and `gres`: Should be equal to the number of GPUs
- `threadspertask`: Divide your threads equally onto all tasks
- `queue` and `account`: SLURM queue and account to use
- `modules`: To be sourced before building and running

Additional JUBE flags can be used to differentiate between the runs:
`jube run benchmark/jube/default.yml --tag=scaling`


### Manual

nekRS can be started according to the following command line, including the previously generated input files.

```
[mpiexec] $NEKRS_HOME/bin/nekrs --backend CUDA --device-id 0 --setup rbc
```

A pre-populated Slurm script can be found as an example in `src/submit.job`.

Using the example submit script, you can start your job in the _case directory_ (see Structure section above), after adjusting it to your configuration. Note that running multiple parallel instances of job can lead to interference between the jobs; we recommend running them in their own directories.

## Verification

The provided benchmark makes a verification and prints a `Verification: Passed` on stdout, when the verification was successful. Additionally, the data used for the verification will be written to the file `vz.csv`.

## Results

The used metric is the total time spent on solving the 600 timesteps, called `total solve time`.

### JUBE

Once all runs are completed, the results can be generated with `jube result -a benchmark/jube/run --style csv`. JUBE will also write the generated table into `benchmark/jube/run/000000/result/result.dat` in CSV format.

### Manual

Each run prints the time for the solve steps to stdout, for example as `total solve time: 120.0s`.

## Commitment

### TCO Baseline

The baseline configuration must be chosen such that the runtime metric (_total solve time_) is less than or equal to 255 s. This value was achieved with 8 nodes with 4 A100 GPUs per node on JUWELS Booster at JSC. Using JUBE, the following output should be gotten:

| nodes | total_solve_time | verification_status |
|-------|------------------|---------------------|
|     8 |          254.773 | Passed              |


### High-Scaling

High-scaling runs probe JUWELS Booster at 642 nodes with a workload to fill the GPU memory of an A100 to 50%, 75%, and 100% respectively (minus a safety margin). The relevant tags are `high_small`, `high_medium`, and `high_large`. We obtain:

|tag| nodes | total_solve_time | verification_status |
|--|-----|------------------|---------------------|
|high_small|   642 |          145.563 |              Passed |
|high_medium|   642 |          216.786 |              Passed |
|high_large|   642 |          277.687 |              Passed |

