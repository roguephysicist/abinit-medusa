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

You would need to compile with stuff like this:

```sh
--with-trio-flavor=netcdf --with-netcdf-libs="-L/home/sma/bin/netcdf/lib -lnetcdf -lnetcdff" --with-netcdf-incs="-I/home/sma/bin/netcdf/include"
```
