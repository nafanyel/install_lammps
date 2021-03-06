# Installing and Running LAMMPS on the HPC Clusters

Below are a set of directions for building CPU and GPU versions of the code on the HPC clusters. LAMMPS can be built in many different ways. You may need to include additional packages when building the executable for your work. The build procedures below are just samples.

The performance of the LAMMPS executable can be greatly improved by including the USER-OMP and USER-INTEL acceleration packages. The [USER-INTEL](https://lammps.sandia.gov/doc/Build_extras.html#user-intel) package takes advantage of our Intel hardware and software. The acceleration arises from mixed-precision arithmetic and vectorization. If mixed-precision arithmetic is valid for your work then we recommend the mixed-precision version of LAMMPS. If not then follow the directions for the double-precision version. Note that one can do [test runs](https://github.com/PrincetonUniversity/install_lammps/tree/master/07_mixed_versus_double) using both versions to see if the results differ substantially.

IMPORTANT: If you are doing biomolecular simulations involving PPPM and the various bond, angle and dihedral potentials, to use the USER-INTEL package you may need these substitutions for the lines in the build procedures below:

```bash
wget https://github.com/lammps/lammps/archive/patch_4Feb2020.tar.gz
module load intel/18.0/64/18.0.3.222
module load intel-mpi/intel/2018.3/64
```

That is, the patched version of the latest release is needed and the Intel 2018 compiler should be used instead of 2019. For more see [this post](https://lammps.sandia.gov/threads/msg85269.html) on the LAMMPS mailing list.

#### Issue with version 29Oct2020

You may encounter the following error:

```
Fatal Error: File 'mpi.mod' opened at (1) is not a GNU Fortran module file
```

The solution is to explicitly specify the Fortran compiler in the .sh build script:

```
-D CMAKE_Fortran_COMPILER=/opt/intel/compilers_and_libraries_2019.5.281/linux/bin/intel64/ifort \
```

The path above corresponds to the module `intel/19.0/64/19.0.5.281`. Use `module show <module name>` to find the path for a different module.


## Obtaining the code and starting the build

The first step is to download the source code and make a build directory (make sure you get the [latest stable release](https://lammps.sandia.gov/download.html)):

```
wget https://github.com/lammps/lammps/archive/stable_29Oct2020.tar.gz
tar zxf stable_29Oct2020.tar.gz
cd lammps-stable_29Oct2020
mkdir build
cd build
```

Note that you may find that some of the tests associated with version 29Oct2020 fail when running `make test`. In this case you should either write to the mailing list or use the previous stable release (wget https://github.com/lammps/lammps/archive/stable_3Mar2020.tar.gz).

The next set of directions vary by cluster. Follow the directions below for the cluster of interest.

## TigerGPU

This cluster is composed of 80 nodes with 28 CPU-cores per node and 4 NVIDIA P100 GPUs per node. TigerGPU should only be used for multi-node jobs that take advantage of GPUs.

#### Mixed-precision version

```
$ ssh <YourNetID>@tigergpu.princeton.edu
$ cd software  # or another directory
$ wget https://raw.githubusercontent.com/PrincetonUniversity/install_lammps/master/01_installing/tigerGpu_user_intel.sh
$ bash tigerGpu_user_intel.sh | tee lammps_mixed.log
```

The LAMMPS build system will add `-qopenmp`, `-restrict` and `-xHost` to the CXX_FLAGS. Note that the build above includes the MOLECULE, RIGID and KSPACE packages. If you do not need these for your simulations then you should remove them from the .sh file after running `wget`.

The following Slurm script can be used to run the job on the TigerGPU cluster:

```
#!/bin/bash
#SBATCH --job-name=lj-melt       # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks=4               # total number of tasks across all nodes
#SBATCH --cpus-per-task=1        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G         # memory per cpu-core (4G is default)
#SBATCH --gres=gpu:1             # number of gpus per node
#SBATCH --time=00:01:00          # total run time limit (HH:MM:SS)

module purge
module load intel/18.0/64/18.0.3.222 intel-mpi/intel/2018.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun $HOME/.local/bin/lmp_tigerGpu -sf gpu -in in.melt
```

The user should vary the various quantities in the Slurm script to find the optimal values.

#### Double-precision version

Connect with: `ssh <YourNetID>@tigergpu.princeton.edu`.

```
# make sure you are on tigergpu.princeton.edu

module purge
module load intel/19.0/64/19.0.5.281 intel-mpi/intel/2019.3/64 cudatoolkit/10.2

# copy and paste the next 6 lines into the terminal
cmake3 -D CMAKE_INSTALL_PREFIX=$HOME/.local -D CMAKE_BUILD_TYPE=Release -D LAMMPS_MACHINE=tigerGpuD \
-D ENABLE_TESTING=yes -D BUILD_MPI=yes -D BUILD_OMP=yes -D CMAKE_C_COMPILER=icc \
-D CMAKE_CXX_COMPILER=icpc -D CMAKE_CXX_FLAGS_RELEASE="-Ofast -xHost -mtune=broadwell -DNDEBUG" \
-D PKG_USER-OMP=yes -D PKG_MOLECULE=yes -D PKG_RIGID=yes \
-D PKG_KSPACE=yes -D FFT=MKL -D FFT_SINGLE=no \
-D PKG_GPU=yes -D GPU_API=cuda -D GPU_PREC=double -D GPU_ARCH=sm_60 -D CUDPP_OPT=yes ../cmake

make -j 10
make test
make install
```

The LAMMPS build system will add `-qopenmp` and `-restrict` to the CXX_FLAGS. Note that the build above includes the MOLECULE, RIGID and KSPACE packages. If you do not need these for your simulations then you should remove them.

The following Slurm script can be used to run the job on the TigerGPU cluster:

```
#!/bin/bash
#SBATCH --job-name=lj-melt       # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks=7               # total number of tasks across all nodes
#SBATCH --cpus-per-task=2        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G         # memory per cpu-core (4G is default)
#SBATCH --gres=gpu:1             # number of gpus per node
#SBATCH --time=00:01:00          # total run time limit (HH:MM:SS)

module purge
module load intel/19.0/64/19.0.5.281 intel-mpi/intel/2019.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
 
srun $HOME/.local/bin/lmp_tigerGpuD -sf gpu -in in.melt.gpu
```

The user should vary the various quantities in the Slurm script to find the optimal values.

Here is a sample LAMMPS script called `in.melt.gpu`:

```
package         gpu 1

units           lj
atom_style      atomic

lattice         fcc 0.8442
region          box block 0 30 0 30 0 30
create_box      1 box
create_atoms    1 box
mass            1 1.0

velocity        all create 1.0 87287

pair_style      lj/cut/gpu 2.5 # explicit gpu pair style
pair_coeff      1 1 1.0 1.0 2.5

neighbor        0.3 bin
neigh_modify    every 20 delay 0 check no

fix             1 all nve
fix             2 all langevin 1.0 1.0 1.0 48279

timestep        0.005

thermo          5000
run             10000
```

To use 2 GPUs, replace `package gpu 1` with `package gpu 2` and `-sf gpu` with `-sf gpu -pk gpu 2` and `#SBATCH --gres=gpu:1` with `#SBATCH --gres=gpu:2`.

## TigerCPU

This cluster is composed of 408 nodes with 40 CPU-cores per node. TigerCPU should be used for running multi-node parallel jobs. Single-node jobs should be run on Della. Connect with: `ssh <NetID>@tigercpu.princeton.edu`.

#### Mixed-precision version

```
$ ssh <YourNetID>@tigercpu.princeton.edu
$ cd software  # or another directory
$ wget https://raw.githubusercontent.com/PrincetonUniversity/install_lammps/master/01_installing/tigerCpu_user_intel.sh
$ bash tigerCpu_user_intel.sh | tee lammps_mixed.log
```

Note that the LAMMPS build system will add `-qopenmp`, `-restrict` and `-xHost` to the CXX_FLAGS.

Below is a sample Slurm script:

```
#!/bin/bash
#SBATCH --job-name=lj-melt       # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks=4              # total number of tasks across all nodes
#SBATCH --cpus-per-task=1        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G         # memory per cpu-core (4G is default)
#SBATCH --time=00:05:00          # total run time limit (HH:MM:SS)

module purge
module load intel/18.0/64/18.0.3.222 intel-mpi/intel/2018.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun $HOME/.local/bin/lmp_tigerCpu -sf intel -in in.melt
```

The user should vary the various quantities in the Slurm script to find the optimal values.

#### Double-precision version

```
# make sure you are on tigercpu.princeton.edu

module purge
module load intel/19.0/64/19.0.5.281 intel-mpi/intel/2019.3/64

# copy and paste the next 4 lines into the terminal
cmake3 -D CMAKE_INSTALL_PREFIX=$HOME/.local -D LAMMPS_MACHINE=tigerCpuD -D ENABLE_TESTING=yes \
-D BUILD_MPI=yes -D BUILD_OMP=yes -D CMAKE_CXX_COMPILER=icpc -D CMAKE_BUILD_TYPE=Release \
-D CMAKE_CXX_FLAGS_RELEASE="-Ofast -xHost -mtune=skylake-avx512 -DNDEBUG" \
-D PKG_USER-OMP=yes -D PKG_MOLECULE=yes ../cmake

make -j 10
make test
make install
```

Note that the LAMMPS build system will add `-qopenmp` and `-restrict` to the CXX_FLAGS.

Below is a sample Slurm script:

```
#!/bin/bash
#SBATCH --job-name=lj-melt       # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks=10              # total number of tasks across all nodes
#SBATCH --cpus-per-task=1        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G         # memory per cpu-core (4G is default)
#SBATCH --time=00:05:00          # total run time limit (HH:MM:SS)

module purge
module load intel/19.0/64/19.0.5.281 intel-mpi/intel/2019.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun $HOME/.local/bin/lmp_tigerCpuD -sf omp -in in.melt
```

The user should vary the various quantities in the Slurm script to find the optimal values.

Here is a sample LAMMPS script called in.melt:

```
units           lj
atom_style      atomic

lattice         fcc 0.8442
region          box block 0 30 0 30 0 30
create_box      1 box
create_atoms    1 box
mass            1 1.0

velocity        all create 1.0 87287

pair_style      lj/cut 2.5
pair_coeff      1 1 1.0 1.0 2.5

neighbor        0.3 bin
neigh_modify    every 20 delay 0 check no

fix             1 all nve
fix             2 all langevin 1.0 1.0 1.0 48279

timestep        0.005

thermo          5000
run             10000
```

## Della

Della is a heterogeneous cluster composed of more than 250 Intel nodes. The microarchitectures of the nodes are Cascade Lake, SkyLake, Broadwell, Haswell, Ivybridge. Della can be used to run a variety of jobs from single-core to parallel, multi-node. The head node features Intel Broadwell CPUs.

#### Mixed-precision version

Run the commands below to build LAMMPS in mixed precision for Della:

```
$ ssh <YourNetID>@della.princeton.edu
$ cd software  # or another directory
$ wget https://raw.githubusercontent.com/PrincetonUniversity/install_lammps/master/01_installing/lammps_mixed_prec_della.sh
$ bash lammps_mixed_prec_della.sh | tee lammps_mixed.log
```

The LAMMPS build system will add `-qopenmp`,  `-restrict` and `-xHost` to the CXX_FLAGS. It is normal to see a large number of messages containing the phrase "has been targeted for automatic cpu dispatch".

The following Slurm script can be used to run the job on Della:

```
#!/bin/bash
#SBATCH --job-name=lj-melt                       # create a short name for your job
#SBATCH --nodes=1                                # node count
#SBATCH --ntasks=16                              # total number of tasks across all nodes
#SBATCH --cpus-per-task=1                        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G                         # memory per cpu-core (4G is default)
#SBATCH --time=00:05:00                          # total run time limit (HH:MM:SS)
#SBATCH --constraint=haswell                     # exclude ivy nodes

module purge
module load intel/18.0/64/18.0.3.222 intel-mpi/intel/2018.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun $HOME/.local/bin/lmp_della -sf intel -in in.melt
```

Users should vary the various quantities in the Slurm script to find the optimal values. If you fail to exclude the Intel Ivy Bridge nodes on Della then you will see an error message like "Please verify that both the operating system and the processor support Intel(R) MOVBE, FMA, BMI, LZCNT and AVX2 instructions."

#### Double-precision version

Run the commands below to build LAMMPS in double precision for Della:

```
$ ssh <YourNetID>@della.princeton.edu
$ cd software  # or another directory
$ wget https://raw.githubusercontent.com/PrincetonUniversity/install_lammps/master/01_installing/lammps_double_prec_della.sh
$ bash lammps_double_prec_della.sh | tee lammps_double.log
```

The LAMMPS build system will add `-qopenmp` and  `-restrict` to the CXX_FLAGS. It is normal to see a large number of messages containing the phrase "has been targeted for automatic cpu dispatch".

The following Slurm script can be used to run the job on Della:

```
#!/bin/bash
#SBATCH --job-name=lj-melt                       # create a short name for your job
#SBATCH --nodes=1                                # node count
#SBATCH --ntasks=16                              # total number of tasks across all nodes
#SBATCH --cpus-per-task=1                        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G                         # memory per cpu-core
#SBATCH --time=00:05:00                          # total run time limit (HH:MM:SS)
#SBATCH --constraint=haswell                     # exclude ivy nodes

module purge
module load intel/19.0/64/19.0.5.281 intel-mpi/intel/2018.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun $HOME/.local/bin/lmp_della_double -sf omp -in in.melt
```

Below is a sample LAMMPS script called `in.melt`:

```
units           lj
atom_style      atomic

lattice         fcc 0.8442
region          box block 0 30 0 30 0 30
create_box      1 box
create_atoms    1 box
mass            1 1.0

velocity        all create 1.0 87287

pair_style      lj/cut 2.5
pair_coeff      1 1 1.0 1.0 2.5

neighbor        0.3 bin
neigh_modify    every 20 delay 0 check no

fix             1 all nve
fix             2 all langevin 1.0 1.0 1.0 48279

timestep        0.005

thermo          5000
run             10000
```

## Traverse

This cluster is quite different from the others given its IBM POWER9 CPUs. Traverse is composed of 46 nodes with 32 physical CPU cores per node and 4 NVIDIA V100 GPUs per node. Users should only be using this cluster if their LAMMPS simulations can use GPUs. The USER-INTEL package cannot be used on Traverse because the CPUs are made by IBM and not Intel.

Run these commands to install LAMMPS on Traverse:

```bash
$ ssh <YourNetID>@traverse.princeton.edu
$ cd software  # or another location
$ wget https://raw.githubusercontent.com/PrincetonUniversity/install_lammps/master/01_installing/traverse.sh
# look at traverse.sh and make modifications if necessary (e.g., add/remove LAMMPS packages)
$ bash traverse.sh | tee build_lammps.log
```

Below is a sample Slurm script to run a simple Lennard-Jones melt:

```
#!/bin/bash
#SBATCH --job-name=lj-melt       # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks=16              # total number of tasks across all nodes
#SBATCH --cpus-per-task=1        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --ntasks-per-core=1      # setting to 1 turns off SMT (max value is 4)
#SBATCH --mem=8G                 # total memory per node (4G is default per cpu-core)
#SBATCH --gres=gpu:1             # number of gpus per node
#SBATCH --time=00:05:00          # total run time limit (HH:MM:SS)

module purge
module load openmpi/gcc/4.0.4/64 cudatoolkit/11.2
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun $HOME/.local/bin/lmp_traverse -sf gpu -in in.melt.gpu
```

Users will need to find the optimal values for `nodes`, `ntasks`, `cpus-per-task`, `ntasks-per-core` and `gres`. Each node of Traverse has 2 CPUs. Each CPU has 16 physical cores. Each physical core has 4 floating point units. A setting of `--ntasks-per-core=4` turns on IBM's simultaneous multithreading (SMT). A setting of `--ntasks-per-core=1` turns it off. Note that the cudatoolkit module does not need to be loaded in the Slurm script since the only CUDA library that the LAMMPS executable depends on is /usr/lib64/libcuda.so, which relates to the driver.

Below is a sample LAMMPS script called `in.melt.gpu`:

```
package         gpu 1

units           lj
atom_style      atomic

lattice         fcc 0.8442
region          box block 0 30 0 30 0 30
create_box      1 box
create_atoms    1 box
mass            1 1.0

velocity        all create 1.0 87287

pair_style      lj/cut/gpu 2.5 # explicit gpu pair style
pair_coeff      1 1 1.0 1.0 2.5

neighbor        0.3 bin
neigh_modify    every 20 delay 0 check no

fix             1 all nve
fix             2 all langevin 1.0 1.0 1.0 48279

timestep        0.005

thermo          5000
run             10000
```

To use 2 GPUs, replace `package gpu 1` with `package gpu 2` and `-sf gpu` with `-sf gpu -pk gpu 2` and `#SBATCH --gres=gpu:1` with `#SBATCH --gres=gpu:2`.

## Perseus

#### Mixed-precision version

```
$ ssh <YourNetID>@perseus.princeton.edu
$ cd software  # or another directory
$ wget https://raw.githubusercontent.com/PrincetonUniversity/install_lammps/master/01_installing/perseus_user_intel.sh
$ bash perseus_user_intel.sh | tee lammps_mixed.log
```

Note that the LAMMPS build system will add `-qopenmp`, `-restrict` and `-xHost` to the CXX_FLAGS.

Below is a sample Slurm script:

```
#!/bin/bash
#SBATCH --job-name=lj-melt       # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks=10              # total number of tasks across all nodes
#SBATCH --cpus-per-task=1        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G         # memory per cpu-core (4G is default)
#SBATCH --time=00:05:00          # total run time limit (HH:MM:SS)
#SBATCH --mail-type=begin        # send email when job begins
#SBATCH --mail-type=end          # send email when job ends
#SBATCH --mail-user=<YourNetID>@princeton.edu

module purge
module load intel/18.0/64/18.0.3.222 intel-mpi/intel/2018.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun $HOME/.local/bin/lmp_perseus -sf intel -in in.melt
```

#### Double-precision CPU version

This cluster is similar to TigerCPU except the CPUs are one generation behind. The directions are exactly the same except for Perseus replace `-mtune=skylake-avx512` with `-mtune=broadwell` and `LAMMPS_MACHINE=tigerCpu` with `LAMMPS_MACHINE=perseus` and use these modules:

```
module load intel/19.0/64/19.0.5.281
module load intel-mpi/intel/2018.3/64
```

## Adroit

Adroit is a heterogeneous cluster with nodes having different microarchitectures. Two of the eighteen nodes have GPUs. A CPU version of LAMMPS on Adroit can be built as follows:

#### Double-precision CPU version

```
module purge
module load intel/19.0/64/19.0.5.281 intel-mpi/intel/2018.3/64

# copy and paste the next 4 lines into the terminal
cmake3 -D CMAKE_INSTALL_PREFIX=$HOME/.local -D LAMMPS_MACHINE=adroit -D ENABLE_TESTING=yes \
-D BUILD_MPI=yes -D BUILD_OMP=yes -D CMAKE_CXX_COMPILER=icpc -D CMAKE_BUILD_TYPE=Release \
-D CMAKE_CXX_FLAGS_RELEASE="-Ofast -DNDEBUG" -D PKG_USER-OMP=yes \
-D PKG_KSPACE=yes -D FFT=MKL -D FFT_SINGLE=yes \
-D PKG_MOLECULE=yes -D PKG_RIGID=yes  ../cmake

make -j 10
make test
make install
```

The LAMMPS build system will add `-qopenmp` and `-restrict` to the CXX_FLAGS.

Below is a sample Slurm script:

```
#!/bin/bash
#SBATCH --job-name=lj-melt       # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks=4               # total number of tasks across all nodes
#SBATCH --cpus-per-task=1        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=1G         # memory per cpu-core (4G is default)
#SBATCH --time=00:05:00          # total run time limit (HH:MM:SS)

module purge
module load intel/19.0/64/19.0.5.281 intel-mpi/intel/2018.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun $HOME/.local/bin/lmp_adroit -sf omp -in in.melt
```

#### Mixed-precision V100 GPU version

```
module purge
module load intel/19.0/64/19.0.5.281 intel-mpi/intel/2018.3/64
module load cudatoolkit/10.1

# copy and paste the next 8 lines into the terminal
cmake3 -D CMAKE_INSTALL_PREFIX=$HOME/.local -D CMAKE_BUILD_TYPE=Release \
-D LAMMPS_MACHINE=adroitGPU -D ENABLE_TESTING=yes -D BUILD_MPI=yes -D BUILD_OMP=yes \
-D CMAKE_C_COMPILER=icc -D CMAKE_CXX_COMPILER=icpc \
-D CMAKE_CXX_FLAGS_RELEASE="-Ofast -mtune=skylake -DNDEBUG" -D PKG_USER-OMP=yes \
-D PKG_MOLECULE=yes -D PKG_RIGID=yes \
-D PKG_KSPACE=yes -D FFT=MKL -D FFT_SINGLE=yes \
-D PKG_GPU=yes -D GPU_API=cuda -D GPU_PREC=mixed -D GPU_ARCH=sm_70 -D CUDPP_OPT=yes \
-D PKG_USER-INTEL=yes -D INTEL_ARCH=cpu -D INTEL_LRT_MODE=threads ../cmake
```

Below is a sample Slurm script:

```
#!/bin/bash
#SBATCH --job-name=lj-melt       # create a short name for your job
#SBATCH --nodes=1                # node count
#SBATCH --ntasks=14              # total number of tasks across all nodes
#SBATCH --cpus-per-task=1        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G         # memory per cpu-core (4G is default)
#SBATCH --gres=gpu:tesla_v100:2  # number of V100 GPUs
#SBATCH --time=00:05:00          # total run time limit (HH:MM:SS)

module purge
module load intel/19.0/64/19.0.5.281 intel-mpi/intel/2018.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

srun $HOME/.local/bin/lmp_adroitGPU -sf gpu -pk gpu 2 -in in.melt.gpu
```

## LAMMPS with Python Interface

LAMMPS can be built with a Python interface as [described here](https://lammps.sandia.gov/doc/Python_head.html). Run the commands below (for Della with USER-INTEL) to build the code in this way:

```
$ ssh <YourNetID>@della.princeton.edu
$ cd software  # or another directory
$ wget https://raw.githubusercontent.com/PrincetonUniversity/install_lammps/master/01_installing/lammps_mixed_prec_python_della.sh
$ bash lammps_mixed_prec_python_della.sh | tee lammps_mixed_python.log
```

To run a parallel job on Della with the Python interface:

```bash
#!/bin/bash
#SBATCH --job-name=lj-melt                       # create a short name for your job
#SBATCH --nodes=1                                # node count
#SBATCH --ntasks=4                               # total number of tasks across all nodes
#SBATCH --cpus-per-task=1                        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G                         # memory per cpu-core (4G is default)
#SBATCH --time=00:05:00                          # total run time limit (HH:MM:SS)
#SBATCH --constraint=haswell|broadwell           # exclude ivy nodes

module purge
module load intel/18.0/64/18.0.3.222 intel-mpi/intel/2018.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/.local/lib64:$HOME/.conda/envs/lammps-env/lib

module load anaconda3/2020.7
conda activate lammps-env

srun python myscript.py
```

Since the command-line switch "-sf intel" cannot be used, one must explicitly turn on the intel package and suffix in `in.lj`:

```
units           lj
atom_style      atomic

package         intel 0
suffix          intel

lattice         fcc 0.8442
region          box block 0 30 0 30 0 30
create_box      1 box
create_atoms    1 box
mass            1 1.0

velocity        all create 1.0 87287

pair_style      lj/cut 2.5
pair_coeff      1 1 1.0 1.0 2.5

neighbor        0.3 bin
neigh_modify    every 20 delay 0 check no

fix             1 all nve
fix             2 all langevin 1.0 1.0 1.0 48279

timestep        0.005

thermo          5000
run             10000
```

Here are the contents of `myscript.py`:

```python
from mpi4py import MPI
from lammps import lammps
lmp = lammps()
lmp.file("in.lj")
me = MPI.COMM_WORLD.Get_rank()
nprocs = MPI.COMM_WORLD.Get_size()
print("Proc %d out of %d procs has" % (me,nprocs),lmp)
MPI.Finalize()
```

See the [package](https://lammps.sandia.gov/doc/package.html) command for more.

To run a parallel job on Della without the Python interface:

```bash
#!/bin/bash
#SBATCH --job-name=lj-melt                       # create a short name for your job
#SBATCH --nodes=1                                # node count
#SBATCH --ntasks=4                               # total number of tasks across all nodes
#SBATCH --cpus-per-task=1                        # cpu-cores per task (>1 if multi-threaded tasks)
#SBATCH --mem-per-cpu=4G                         # memory per cpu-core (4G is default)
#SBATCH --time=00:05:00                          # total run time limit (HH:MM:SS)
#SBATCH --constraint=haswell|broadwell           # exclude ivy nodes

module purge
module load intel/18.0/64/18.0.3.222 intel-mpi/intel/2018.3/64
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/.local/lib64:$HOME/.conda/envs/lammps-env/lib

srun $HOME/.local/bin/lmp_della -sf intel -in in.melt
```

Here are some timings for the system above with and without the user-intel package on the cascade lake nodes of Della with 4 MPI processes:

| time (s) | user-intel |
|:------:|:------:|
| 120 | with |
| 190 | without |

## Using make

The procedure below can be used to build LAMMPS with USER-INTEL using make instead cmake. This provides more control over the build. It makes an executable with AVX512 instructions and is not specific to Cascade Lake processors.

```
wget https://github.com/lammps/lammps/archive/stable_3Mar2020.tar.gz
tar zxf stable_3Mar2020.tar.gz 
cd lammps-stable_3Mar2020/
cd src
make ps
make yes-user-intel
make yes-kspace
make yes-rigid
make yes-user-omp
make yes-molecule
make yes-misc
module load intel/19.1/64/19.1.1.217 intel-mpi/intel/2019.7/64
mkdir -p MAKE/MINE
cd MAKE/MINE
wget https://raw.githubusercontent.com/PrincetonUniversity/install_lammps/master/01_installing/Makefile.cascade
cd ../..
make cascade
```

## VMD Plugins

Below is a procedure for using the `dump` command with the `molfile` style (be sure to replace `<YourNetID>` twice): 

```
mkdir -p software/vmd_precompiled
cd software/vmd_precompiled
wget --no-check-certificate https://www.ks.uiuc.edu/Research/vmd/vmd-1.9.3/files/final/vmd-1.9.3.bin.LINUXAMD64-CUDA8-OptiX4-OSPRay111p1.opengl.tar.gz
tar zxf vmd-1.9.3.bin.LINUXAMD64-CUDA8-OptiX4-OSPRay111p1.opengl.tar.gz
```

Next build LAMMPS:

```
wget https://github.com/lammps/lammps/archive/patch_4Feb2020.tar.gz
tar -zxvf patch_4Feb2020.tar.gz 
cd lammps-patch_4Feb2020
mkdir build && cd build

module purge
module load intel/18.0/64/18.0.3.222
module load intel-mpi/intel/2018.3/64

# below the resulting executable is set to be lmp_perseus (you may want to rename it)

# copy and paste the following into the terminal
cmake3 -D CMAKE_INSTALL_PREFIX=$HOME/.local -D LAMMPS_MACHINE=perseus -D ENABLE_TESTING=yes \
-D BUILD_MPI=yes -D BUILD_OMP=yes -D CMAKE_CXX_COMPILER=icpc -D CMAKE_BUILD_TYPE=Release \
-D CMAKE_CXX_FLAGS_RELEASE="-Ofast -mtune=broadwell -DNDEBUG" \
-D PKG_USER-OMP=yes -D PKG_MOLECULE=yes -D PKG_RIGID=yes -D PKG_MISC=yes \
-D PKG_KSPACE=yes -D FFT=MKL -D FFT_SINGLE=yes \
-D MOLFILE_INCLUDE_DIRS=/home/<YourNetID>/software/vmd_precompiled/vmd-1.9.3/plugins/include -D PKG_USER-MOLFILE=yes \
-D PKG_USER-INTEL=yes -D INTEL_ARCH=cpu -D INTEL_LRT_MODE=threads ../cmake

make -j 10
make test
make install
```

Finally, run a test:

```
units           lj
atom_style      atomic

lattice         fcc 0.8442
region          box block 0 30 0 30 0 30
create_box      1 box
create_atoms    1 box
mass            1 1.0

velocity        all create 1.0 87287

pair_style      lj/cut 2.5
pair_coeff      1 1 1.0 1.0 2.5

neighbor        0.3 bin
neigh_modify    every 20 delay 0 check no

fix             1 all nve
fix             2 all langevin 1.0 1.0 1.0 48279

dump            mf1 all molfile 100 melt-*.pdb pdb /home/<YourNetID>/software/vmd_precompiled/vmd-1.9.3/plugins/LINUXAMD64/molfile

timestep        0.005

thermo          500
run             1000
```

## Getting Help

If you encounter any difficulties while working with LAMMPS then please send an email to <a href="mailto:cses@princeton.edu">cses@princeton.edu</a> or attend a [help session](https://researchcomputing.princeton.edu/education/help-sessions).
