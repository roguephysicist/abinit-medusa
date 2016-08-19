Instructions for compiling and installing ABINIT 7.10.5
=======================================================

A full ABINIT install would optimally include support for the following
libraries or features:

|   Component   |   Package             |
|---------------|-----------------------|
|   Parallel    |   Intel MPI           |
|   LinAlg      |   Intel MKL           |
|   FFT         |   FFTW or dfti        |
|   TRIO        |   NetCDF + ETSF_IO    |

Intel MPI and MKL are already installed on the system. They will be referenced
to during the configuration stage so it is important to have those paths
configured correctly. Refer to your system documentation for more information.


Considerations for compiling and installing on Medusa
-------------------------------------------------------

We will make frequent use of the `module` tool that takes care setting up the
proper paths and libraries for us. In general, all compiler flags will generally
look like

```sh
XXFLAGS="-axCORE-AVX2,-axSSE4.2 -ip -shared-intel -mcmodel=small -fp-model precise"
```

which takes advantage of the special processor instructions available for
Medusa. Binaries compiled with these flags will be compatible across all three
platforms. That being said, sometimes software will refuse to compile properly
so you should always check these flags and run tests after compiling.

????Once Pedro finishes configuring the software packages and the `module` tool,
I will talk about that here.????

Define an install PATH

```sh
export INSTALL_PRE=$HOME/bin
export COMP_FLAGS="-O3"
#export COMP_FLAGS="-axCORE-AVX2,-axSSE4.2 -ip -shared-intel -mcmodel=small -fp-model precise"
```

Compiling FFTW (fftw-3.3.4) 
-------------------------------------------------------

FFTW needs to be compiled with the same compilers that you will use to compile
ABINIT. ABINIT needs both float and non-float FFTW libraries. We can compile
these with

```sh
./configure --prefix=${INSTALL_PRE}/fftw-3.3.4 CC=icc F77=ifort MPICC=mpiicc --enable-mpi --enable-fma CFLAGS=${COMP_FLAGS} FFLAGS=${COMP_FLAGS} && make && make install
```

and repeating the same instruction again after adding the `--enable-float`
option.


Compiling ABINIT 7.10.5 
-------------------------------------------------------

I highly suggest running `make clean` in the ABINIT source directory before
attempting to compile. For a truly clean build, you run `./wipeout.sh` to
totally remove all generated files. If you do this, you will need to run
`./autogen.sh` to regenerate the configure script and Makefiles.

If you use the `medusa.ac` file that is included in the `abinit-7.10.5`
directory, you can simple run 

```sh
./configure && make multi multi_nprocs=16 && make install
```

or if you want to configure from the command line, the following is equivalent
to the `medusa.ac` configuration file (remember to ignore the config file)

```sh
./configure --prefix=/home/sma/test/abinit-7.10.5 --enable-64bit-flags --enable-optim CC=mpiicc CFLAGS_OPTIM="${COMP_FLAGS}" CXX=mpiicpc CXXFLAGS_OPTIM="${COMP_FLAGS}" FC=mpiifort F77=mpiifort FCFLAGS_OPTIM="${COMP_FLAGS}" --enable-macroave --enable-mpi --enable-mpi-io --with-mpi-level=2 MPI_RUNNER=mpiexec.hydra --enable-connectors --with-fft-flavor="fftw3" --with-fft-incs="-I${INSTALL_PRE}/fftw-3.3.4/include" --with-fft-libs="-L${INSTALL_PRE}/fftw-3.3.4/lib -lfftw3 -lfftw3f -lfftw3_mpi -lfftw3f_mpi" --enable-fallbacks fcflags_opt_43_ptgroups="-g -O0" && make multi multi_nprocs=16 && make install
```
It is worth mentioning that the `fcflags_opt_43_ptgroups="-g -O0"` option
prevents optimization to the code in group 43, which is extremely time
consuming. Building in this manner takes around 72 minutes using 16 cores. On
the other hand, excluding the `fcflags_opt_43_ptgroups="-g -O0"` option
increases build time to 146 minutes with the same 16 cores. I highly suggest
using the mentioned build option.


Unfortunately, I have not succesfully integrated NetCDF, ETSF IO, or Intel MKL
support yet. The following options are still works in progress and will be
eventually migrated as I get them to work.

```sh
--enable-bse-unpacked --enable-gw-dpc --enable-vdwxc --with-trio-flavor=netcdf --with-netcdf-libs="-L/home/sma/bin/netcdf/lib -lnetcdf -lnetcdff" --with-netcdf-incs="-I/home/sma/bin/netcdf/include" --with-linalg-flavor=mkl --with-linalg-incs=-I/opt/intel/compilers_and_libraries_2016.1.150/linux/mkl/include/intel64/ilp64  --with-linalg-libs="-L/opt/intel/compilers_and_libraries_2016.1.150/linux/mkl/lib/intel64 -lmkl_scalapack_ilp64 -lmkl_intel_ilp64 -lmkl_intel_thread -lmkl_core -lmkl_blacs_intelmpi_ilp64" FCFLAGS=-mklcluster
```


Works in progress
-------------------------------------------------------

The following packages compile and install correctly. However, compiling ABINIT
with them does not seen to enable their functionality. All of these packages
should be tested with either `make check` or `make test` after compiling. You
should enable an MPI ring by issuing `mpdboot` before running the tests.
Remember to close the ring with `mpdallexit` after completing the tests. If they
do not pass the tests, then you must stop and correct the problem before
continuing with other packages.

You will also note that I install the three NetCDF libraries to the same
directory. This is just to keep the paths simple, as all the NetCDF libs and
include files will be in the same location.


### zlib (zlib-1.2.8)
zlib is a free lossless data-compression library. It is a dependency of HDF5.
```sh
export CC=icc; export CFLAGS="${COMP_FLAGS}"; ./configure --prefix="/home/sma/bin/zlib-1.2.8" && make && make install
```


### HDF5 (hdf5-1.8.16)
HDF5 is a data model, library, and file format for storing and managing data. It
is a dependency of NetCDF.
```sh
./configure --prefix=/home/sma/bin/hdf5-1.8.16 CC=mpiicc FC=mpiifort F77=mpiifort CXX=mpiicpc --enable-fortran --enable-parallel --with-zlib="/home/sma/bin/zlib-1.2.8" CFLAGS="${COMP_FLAGS}" CXXFLAGS="${COMP_FLAGS}" FCFLAGS="${COMP_FLAGS}" && make && make install
```


### Parallel NetCDF (parallel-netcdf-1.6.1)

PnetCDF is a library providing high-performance parallel I/O while still
maintaining file-format compatibility with Unidata's NetCDF. PnetCDF must be
compiled before NetCDF and does not require HDF5.
```sh
./configure --prefix=/home/sma/bin/netcdf MPICC=mpiicc MPICXX=mpiicpc MPIF77=mpiifort MPIF90=mpiifort CC=icc CXX=icpc FC=ifort F77=ifort  CFLAGS="${COMP_FLAGS} -fPIC" CXXFLAGS="${COMP_FLAGS} -fPIC" FCFLAGS="${COMP_FLAGS} -fPIC" FFLAGS="${COMP_FLAGS} -fPIC" && make && make install
```


### NetCDF C (netcdf-4.3.3.1)

NetCDF is a set of software libraries and self-describing, machine-independent
data formats that support the creation, access, and sharing of array-oriented
scientific data. This is the standard C library. It requires HDF5 and PnetCDF to
build as indicated below. You should also review the [documentation]
(https://www.unidata.ucar.edu/software/netcdf/netcdf-4/newdocs/netcdf-install/Quick-Instructions.html).
```sh
./configure --prefix=/home/sma/bin/netcdf CC=mpiicc --enable-netcdf-4 --enable-pnetcdf --enable-parallel-tests --enable-large-file-tests CFLAGS="-I/home/sma/bin/hdf5-1.8.16/include -I/home/sma/bin/netcdf/include ${COMP_FLAGS}" LDFLAGS="-L/home/sma/bin/hdf5-1.8.16/lib -L/home/sma/bin/netcdf/lib" CPPFLAGS="-I/home/sma/bin/hdf5-1.8.16/include -I/home/sma/bin/netcdf/include" && make && make install
```


### NetCDF Fortran (netcdf-fortran-4.4.2)

See the [documentation]
(http://www.unidata.ucar.edu/software/netcdf/docs/netcdf-fortran-install.html).
```sh
export LD_LIBRARY_PATH=/home/sma/bin/netcdf/lib:${LD_LIBRARY_PATH}; ./configure --prefix=/home/sma/bin/netcdf CC=mpiicc FC=mpiifort F77=mpiifort --enable-parallel-tests --enable-extra-tests --enable-extra-example-tests CPPFLAGS="-I/home/sma/bin/netcdf/include" LDFLAGS="-L/home/sma/bin/netcdf/lib" CFLAGS="-I/home/sma/bin/netcdf/include ${COMP_FLAGS}" FCFLAGS="-I/home/sma/bin/netcdf/include ${COMP_FLAGS}" FFLAGS="-I/home/sma/bin/netcdf/include ${COMP_FLAGS}" && make && make install
```


### ETSF IO

I think that this is currently not working. Also, I have seen conflicting
reports that say that this package is deprecated and should not be used, while
others say that it is necesarry for using ABINIT with NetCDF.
```sh
./configure --prefix=/home/sma/bin/etsf_io-1.0.4 CC=mpiicc FC=mpiifort F77=mpiifort CXX=mpiicpc CFLAGS="${COMP_FLAGS}" CXXFLAGS="${COMP_FLAGS}" FCFLAGS="${COMP_FLAGS}" --with-netcdf-libs="-L/home/sma/bin/netcdf-4.3.3.1/lib -lnetcdff -lnetcdf -Wl,-rpath=/home/sma/bin/netcdf-4.3.3.1/lib" --with-netcdf-incs="-I/home/sma/bin/netcdf-4.3.3.1/include"
```
