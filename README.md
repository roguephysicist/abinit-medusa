Instructions for compiling and installing ABINIT 7.10.5
=======================================================

A full ABINIT install would optimally include support for the following
libraries or features:

|   Component   |   Package             |
|---------------|-----------------------|
|   Parallel    |   Intel MPI           |
|   LinAlg      |   Intel MKL           |
|   FFT         |   FFTW or dfti        |
|   TRIO (N/A)  |   NetCDF + ETSF_IO    |

Intel MPI and MKL are already installed on the system. They will be referenced
to during the configuration stage so it is important to have those paths
configured correctly. Refer to your system documentation for more information.


Considerations for compiling and installing on Medusa
-------------------------------------------------------

In general, compiler flags for this system should look something like

```sh
COMP_FLAGS="-axCORE-AVX2,-axSSE4.2 -ip -shared-intel -mcmodel=small -fp-model precise"
```

which takes advantage of the special processor instructions available for
Medusa. Binaries compiled with these flags will be compatible across all three
platforms. That being said, sometimes software will refuse to compile properly
so you should always check these flags and run tests after compiling. I have
tested ABINIT compiled with and without these specialized flags, and saw no
speed benefit whatsoever. Therefore, it is best to compile with the most basic
options in order to troubleshoot any problems that may arise during the
compilation. Something like `COMP_FLAGS="-O3"` will suffice.

Note that when using the `module load` command, sometimes default variables get
set that may override your own configuration options. These are easy to unset
with

```
CC=""; CXX=""; FC=""; F77=""
```

It is convenient to define an install PATH and you default compile options from
the start,

```sh
export INSTALL_PRE=/opt/science/bin
export COMP_FLAGS="-O3"
```

Compiling FFTW (fftw-3.3.5) 
-------------------------------------------------------

FFTW needs to be compiled with the same compilers that you will use to compile
ABINIT. ABINIT needs both float and non-float FFTW libraries. We can compile
these with

```sh
./configure --prefix=${INSTALL_PRE}/fftw-3.3.5-intel16.2.181 CC=icc F77=ifort MPICC=mpiicc --enable-mpi --enable-fma CFLAGS=${COMP_FLAGS} FFLAGS=${COMP_FLAGS} && make && make install
```

and repeating the same instruction again after adding the `--enable-float`
option.


Compiling ABINIT 8.0.8 
-------------------------------------------------------

I highly suggest running `make clean` in the ABINIT source directory before
attempting to compile. For a truly clean build, you run `./wipeout.sh` to
totally remove all generated files. If you do this, you will need to run
`./autogen.sh` to regenerate the configure script and Makefiles.

If you use the included `medusa.ac` file, you can simply run 

```sh
./configure && make multi multi_nprocs=16 && make install
```

It is worth mentioning that the `fcflags_opt_43_ptgroups="-g -O0"` option
prevents optimization to the code in group 43, which is extremely time
consuming. Building in this manner takes around 72 minutes using 16 cores. On
the other hand, excluding the `fcflags_opt_43_ptgroups="-g -O0"` option
increases build time to 146 minutes with the same 16 cores. 

You can use the `tests/runtests.py` program for thoroughly testing your new
ABINIT build. You can run this from any directory. For the full battery of
tests, run

```sh
/opt/science/src/abinit-8.0.8/tests/runtests.py -j4 -n4
```

The `-j4` options indicates that you are using 4 python threads, which will run
4 tests concurrently. The `-n4` option allows for 4 MPI threads, that will be
used on tests that have parallelization enabled. This will allow 16 threads max
to be used (4 python threads * 4 MPI threads). The more python threads you run,
the faster the entire battery of tests will finish. More MPI threads are
typically not necessary, save for very specific tests that may ask for more. As
we did not build or include many of the special features in ABINIT, many tests
will be skipped. You can run specific series of tests like

```sh
tests/runtests.py -j4 -n4 paral
```

to test specific features. Refer to the oficial ABINIT documentation for more
information about each series. Lastly, the results of the tests can be viewed
with the HTML report produced inside the `Test_suite` directory.
