# Running ORCHIDEE on a New PC or Server

This repository documents a tested procedure for compiling and running ORCHIDEE on a local Ubuntu machine using GCC and Spack.

The installation has been tested with:

- Ubuntu 24.04.3
- GCC 13.3.0
- Spack v0.23.1
- OpenMPI 4.0.7
- HDF5 1.10.7
- NetCDF-C 4.7.4
- NetCDF-Fortran 4.5.3

The main goal is to avoid version conflicts between MPI, HDF5, NetCDF, IOIPSL, XIOS, and ORCHIDEE by using one consistent software stack.

## Repository layout

```text
.
├── README.md
└── arch
    ├── orchidee
    │   ├── arch-gcc_pc.env
    │   ├── arch-gcc_pc.fcm
    │   └── arch-gcc_pc.path
    ├── ioipsl
    │   ├── arch-gcc_pc.fcm
    │   └── arch-gcc_pc.path
    └── xios
        ├── arch-gcc_pc.fcm
        └── arch-gcc_pc.path
```

The original Word draft is intentionally not tracked because it may contain local notes or access credentials.

## 1. Install Ubuntu and GCC

Ubuntu 24.04.3 is recommended because it provides GCC 13.3.0 by default. This avoids the need to install a separate GCC version.

If `gfortran` is not available, install it with:

```bash
sudo apt update
sudo apt install gfortran
```

Check the compiler version:

```bash
gcc --version
gfortran --version
```

## 2. Install Spack

Install Git if needed:

```bash
sudo apt install git
```

Clone the tested Spack release:

```bash
git clone -b v0.23.1 https://github.com/spack/spack.git ~/spack
source ~/spack/share/spack/setup-env.sh
```

Detect the compiler:

```bash
spack compiler find
spack compiler info gcc@13.3.0
```

The output should look similar to:

```text
gcc@13.3.0:
    paths:
        cc  = /usr/bin/gcc
        cxx = /usr/bin/g++
        f77 = /usr/bin/gfortran
        fc  = /usr/bin/gfortran
    modules = []
    operating system = ubuntu24.04
```

## 3. Install OpenMPI, HDF5, and NetCDF

Install the required libraries in this order:

```bash
spack install openmpi@4.0.7 %gcc@13
spack install hdf5@1.10.7 +hl +mpi %gcc@13 ^openmpi@4.0.7
spack install netcdf-c@4.7.4 +mpi %gcc@13 ^hdf5@1.10.7 ^openmpi@4.0.7
spack install netcdf-fortran@4.5.3 %gcc@13 ^netcdf-c@4.7.4
```

Load the environment:

```bash
source ~/spack/share/spack/setup-env.sh
spack load openmpi@4.0.7
spack load hdf5@1.10.7
spack load netcdf-c@4.7.4
spack load netcdf-fortran@4.5.3
```

If there are multiple matching Spack installations, use `spack find -l <package>` and load the package with its hash.

## 4. Download ORCHIDEE

Clone `modipsl` and download ORCHIDEE:

```bash
git clone https://gitlab.in2p3.fr/ipsl/icmc/plateforme/modipsl/modipsl.git modipsl
cd modipsl/util
./model ORCHIDEE_trunk
```

Some ORCHIDEE or IPSL resources may require institutional credentials. Do not commit usernames, passwords, or access tokens to this repository.

## 5. Modify arch files

Create a local `gcc_pc` architecture for ORCHIDEE, IOIPSL, and XIOS.

### ORCHIDEE

Copy the ORCHIDEE arch files:

```bash
cp arch/orchidee/arch-gcc_pc.env ~/modipsl/modeles/ORCHIDEE/config_offline/ARCH/
cp arch/orchidee/arch-gcc_pc.fcm ~/modipsl/modeles/ORCHIDEE/config_offline/ARCH/
cp arch/orchidee/arch-gcc_pc.path ~/modipsl/modeles/ORCHIDEE/config_offline/ARCH/
```

### IOIPSL

Copy the IOIPSL arch files:

```bash
cp arch/ioipsl/arch-gcc_pc.fcm ~/modipsl/modeles/IOIPSL/arch/
cp arch/ioipsl/arch-gcc_pc.path ~/modipsl/modeles/IOIPSL/arch/
```

### XIOS

Copy the XIOS arch files:

```bash
cp arch/xios/arch-gcc_pc.fcm ~/modipsl/modeles/XIOS/arch/
cp arch/xios/arch-gcc_pc.path ~/modipsl/modeles/XIOS/arch/
```

If your local ORCHIDEE checkout uses different arch directories, copy the files to the matching `ARCH` or `arch` directory in your checkout.

## 6. Compile and test ORCHIDEE

Go to the ORCHIDEE offline configuration directory:

```bash
cd ~/modipsl/modeles/ORCHIDEE/config_offline/
```

Compile with:

```bash
./compile_orchidee_ol.sh -arch gcc_pc
```

After a successful compilation, ORCHIDEE can be launched directly with:

```bash
./orchideedriver
```

## 7. Optional: avoid specifying `-arch` every time

If you do not want to manually pass `-arch gcc_pc` for each compilation, edit `compile_orchidee_ol.sh` and add your hostname to the architecture detection block:

```bash
if [ $fcm_arch == default ] ; then
    case $( hostname -s ) in
        jean-zay*)
            fcm_arch=X64_JEANZAY;;
        your-hostname)
            fcm_arch=gcc_pc;;
        *)
            echo Current host is not known. You must use option -arch to specify which architecture files to use.
            echo Exit now.
            exit
    esac
fi
```

Replace `your-hostname` with the output of:

```bash
hostname -s
```

## 8. Notes on compilation issues

IOIPSL and XIOS may compile without major changes, but ORCHIDEE itself can fail with GCC because GCC is stricter than Intel compilers used on some institutional systems.

When compilation errors occur:

1. Read the compiler error message carefully.
2. Identify the source file and line reported by the compiler.
3. Modify the corresponding source code if the error is caused by GCC compatibility.
4. Recompile until the build completes.

## 9. Running ORCHIDEE with libIGCM

ORCHIDEE can be run directly with `orchideedriver`, but using libIGCM is recommended for a more standardized simulation workflow.

For a local machine that is not included in the official libIGCM system list, edit:

```text
~/libIGCM/libIGCM_sys/libIGCM_sys_default.ksh
```

Set the output directory, for example:

```bash
typeset -r RUN_DIR_PATH=${RUN_DIR_PATH:=/home_local/${LOGIN}/RUN_DIR/tmp$$}
```

Set the temporary run directory, for example:

```bash
typeset -r RUN_DIR_PATH=${RUN_DIR_PATH:=/home/scratch01/$LOGIN/RUN_DIR/$PBS_O_LOGNAME.$PBS_JOBID}
```

Adjust these paths to match the local storage layout of your machine or server.

## Troubleshooting

- Make sure Spack is sourced before compiling.
- Make sure the same OpenMPI, HDF5, NetCDF-C, and NetCDF-Fortran stack is used everywhere.
- If `spack load` is ambiguous, load packages with their hashes.
- If NetCDF headers or libraries are not found, check `spack location -i netcdf-c@4.7.4` and `spack location -i netcdf-fortran@4.5.3`.
- If MPI wrappers are not found, check `which mpif90`, `which mpicxx`, and `mpif90 --showme`.

## Credits

Prepared by Lei Zhu, with help from Guillaume Marie.
