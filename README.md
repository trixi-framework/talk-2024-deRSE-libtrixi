# deRSE24: Controlling parallel simulations in Julia from C/C++/Fortran programs with libtrixi

[![License: MIT](https://img.shields.io/badge/License-MIT-success.svg)](https://opensource.org/licenses/MIT)

This is the companion repository for the talk

**Controlling parallel simulations in Julia from C/C++/Fortran programs with libtrixi**  
[*Michael Schlottke-Lakemper*](https://lakemper.eu), [*Benedict Geihe*](https://www.mi.uni-koeln.de/NumSim/dr-benedict-geihe/)  
[University of Würzburg, HS3, 6th March 2024, 11:30 AM](https://events.hifis.net/event/994/contributions/7970/)  

(see abstract [below](#abstract)). The talk is part of the session
[Cross-Platform Development with C/C++](https://events.hifis.net/event/994/sessions/2730) at
[*deRSE24 - Conference for Research Software Engineering in Germany*](https://go.uniwue.de/derse24),
which takes place in Würzburg, Germany from 5th to 7th March 2024.

Besides the presentations slides ([talk-2024-deRSE-libtrixi.pdf](talk-2024-deRSE-libtrixi.pdf)),
this repository contains instructions and files that allow one to
[reproduce](#reproducibility) (some of) the results in the talk.

## Abstract
The Julia programming language aims to provide a modern approach to scientific high-performance
computing by combining a high-level, accessible syntax with the runtime efficiency of traditional
compiled languages. Due to its native ability to call C and Fortran functions, Julia often acts as a
glue code in multi-language projects, enabling the reuse of existing libraries implemented in
C/Fortran. With the software library [libtrixi](https://github.com/trixi-framework/libtrixi), we
reverse this workflow: It allows one to control
[Trixi.jl](https://github.com/trixi-framework/Trixi.jl), a complex Julia package for parallel,
adaptive numerical simulations, from a main program written in C/C++/Fortran. In this talk, we will
present the overall design of libtrixi, show some of the challenges we had to overcome, and discuss
continuing limitations. Furthermore, we will provide some insights into the Julia C API and into the
PackageCompiler.jl project for static compilation of Julia code. Besides the implications for our
specific use case, these experiences can serve as a foundation for other projects that aim to
integrate Julia-based libraries into existing code environments, opening up new avenues for
sustainable software workflows.


## Reproducibility

The instructions are written based on a x64 machine with Ubuntu Linux 22.04, on which the results
used in the talk were obtained as well.

### Getting started

The following standard software packages need to be made available locally before installing
[libtrixi](https://github.com/trixi-framework/libtrixi):
* C compiler with support for C11 or later (e.g., [GCC](https://gcc.gnu.org/) or [Clang](https://clang.llvm.org/))
  (we used GCC v10.5)
* Fortran compiler with support for Fortran 2018 or later (e.g., [Gfortran](https://gcc.gnu.org/fortran/))
  (we used Gfortran v10.5)
* [CMake](https://cmake.org/)
  (we used cmake v3.16.3)
* MPI (e.g., [Open MPI](https://www.open-mpi.org/) or [MPICH](https://www.mpich.org/))
  (we used OpenMPI v3.1)
* [HDF5](https://www.hdfgroup.org/solutions/hdf5/)
  (we used hdf5-openmpi v1.10)
* [Julia](https://julialang.org/downloads/platform/) v1.10
  (it is advised to use tarballs from the official website or the `juliaup` manager, we used v1.10.0)
* [Paraview](https://paraview.org) for visualization
  (we used v5.11.2)

The following software packages require manual compilation and installation. To simplify
the installation instructions, we assume that the environment variable `WORKDIR` was set to the
directory, where all the code is downloaded to and compiled, and `PREFIX` was set to an installation
prefix. For example,
```shell
export WORKDIR=$HOME/repro-talk-2024-deRSE-libtrixi
export PREFIX=$WORKDIR/install
```


#### t8code

t8code is a meshing library which is used in `Trixi.jl`. The source code can be obtained from the official
[website](https://github.com/DLR-AMR/t8code). Here is an example for compiling and installing it (we used v1.6.1):

```shell
cd $WORKDIR
git clone --branch 'v1.6.1' --depth 1 https://github.com/DLR-AMR/t8code.git
cd t8code
git submodule init
git submodule update
./bootstrap
./configure --prefix="${PREFIX}/t8code" \
            CFLAGS="-Wall -O0 -g" \
            CXXFLAGS="-Wall -O0 -g" \
            --enable-mpi \
            --enable-debug \
            CXX=mpicxx \
            CC=mpicc \
            --enable-static \
            --without-blas \
            --without-lapack
make
make install
```


#### libtrixi

Here is an example for compiling and installing `libtrixi` (we used v0.1.4):

```shell
git clone --branch 'v0.1.5' --depth 1 https://github.com/trixi-framework/libtrixi.git
cd libtrixi
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=${HOME}/install/libtrixi ..
make
make install
```

A separate Julia project for use with libtrixi has to be set up:

```shell
mkdir ${HOME}/install/libtrixi-julia
cd ${HOME}/install/libtrixi-julia
${HOME}/install/libtrixi/bin/libtrixi-init-julia \
  --t8code-library ${HOME}/install/t8code/lib/libt8.so \
  --hdf5-library /usr/lib/x86_64-linux-gnu/hdf5/openmpi/libhdf5.so \
  ${HOME}/install/libtrixi
```

To exactly reproduce the results presented in the talk, the provided [Manifest.toml](Manifest.toml)
can be used to install the same package versions for all Julia dependencies:

```shell
cp Manifest.toml ${HOME}/install/libtrixi-julia/
cd ${HOME}/install/libtrixi-julia
JULIA_DEPOT_PATH=./julia-depot julia --project=.
julia> import Pkg
julia> Pkg.instantiate()
```


#### Trixi2Vtk

If the provided `Manifest.toml` was not used (see above),
[Trixi2Vtk](https://github.com/trixi-framework/Trixi2Vtk.jl)
still needs to be installed for later post processing:

```shell
cd ${HOME}/install/libtrixi-julia
JULIA_DEPOT_PATH=./julia-depot julia --project=.
julia> import Pkg
julia> Pkg.add("Trixi2Vtk")
```


### Running the examples

#### Rising thermal perturbation

```shell
cd ${HOME}/install/libtrixi
mpirun -n 2 ./bin/trixi_controller_simple_f \
  ${HOME}/install/libtrixi-julia \
  ${HOME}/install/libtrixi/share/libtrixi/LibTrixi.jl/examples/libelixir_tree2d_warm_bubble.jl
```

Output files will be written to `${HOME}/install/libtrixi/out_bubble` and need to be post processed:

```shell
cd ${HOME}/install/libtrixi/out_bubble
JULIA_DEPOT_PATH=${HOME}/install/libtrixi-julia/julia-depot \
   julia --project=${HOME}/install/libtrixi-julia
julia> using Trixi2Vtk
julia> trixi2vtk("solution*.h5")
```

The results can now be viewed using, e.g, paraview.
We used [Tpotpert_contour.pvsm](Tpotpert_contour.pvsm) and [Tpotpert.pvsm](Tpotpert.pvsm)
to generate our results.


#### Baroclinic instability

```shell
cd ${HOME}/install/libtrixi
mpirun -n 12 ./bin/trixi_controller_simple_f \
  ${HOME}/install/libtrixi-julia \
  ${HOME}/install/libtrixi/share/libtrixi/LibTrixi.jl/examples/libelixir_p4est3d_euler_baroclinic_instability.jl
```

Output files will be written to `${HOME}/install/libtrixi/out_baroclinic` and need to be post processed:

```shell
cd ${HOME}/install/libtrixi/out_bubble
JULIA_DEPOT_PATH=${HOME}/install/libtrixi-julia/julia-depot \
   julia --project=${HOME}/install/libtrixi-julia
julia> using Trixi2Vtk
julia> trixi2vtk("solution*.h5")
```

The results can now be viewed using, e.g, paraview.
We used [pressure.pvsm](pressure.pvsm) to generate our results.


## Authors
This repository was initiated by
[Benedict Geihe](https://www.mi.uni-koeln.de/NumSim/dr-benedict-geihe/)
and [Michael Schlottke-Lakemper](https://lakemper.eu).


## License
The contents of this repository are licensed under the MIT license (see [LICENSE.md](LICENSE.md)).
