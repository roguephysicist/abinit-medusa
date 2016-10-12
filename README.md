Instructions for building ABINIT on Medusa
=======================================================

This is a detailed guide for building highly optimized binaries for ABINIT on
Medusa. We will make extensive use of the Intel MPI and MKL libraries. This
streamlines the process considerably; combined with processor-specific
optimization, the resulting binaries are are extremely fast without sacrificing
accuracy.

As the Intel MPI and MKL libraries are already installed and optimized for the
system, we do not need to build any intermediate software. If you choose not to
use these libraries, then you will have to build your own external libraries for
FFTW, LINALG, and MPI, or default to the internal connectors that come with
ABINIT. However, this is outside the scope of this guide.


Procedure
-------------------------------------------------------

We will use the adjoining `abinit-x.x.x.ac` configure files. Choose the
appropriate one for your version of ABINIT. This file will greatly facilitate
the build process as it contains all the correct build options and paths. Review
this file carefully, as it contains a lot of useful information. If you wish to
add more connectors or experiment with other options, I recommend doing so using
this file as most of the variables are extensively documented.

**Typically, the only thing you will need to modify is the `prefix`. Set this to
wherever you want to install ABINIT.**


### Step 0 - Prepare the build environment

You can compile from any node in Medusa. The appropriate environment should be
loaded by default for common users in the `medusa`, `fat`, and `hexa` nodes.
Verify this by checking your `$PATH`. You should see the paths for the compiler
binaries (`.../linux/bin/intel64`) and the MPI binaries
(`.../linux/mpi/intel64/bin)`. Verify that you are able to execute `mpiifort`
directly. Lastly, verify that the `${MKLROOT}` and `${I_MPI_ROOT}` paths are
correct, and match those in your `$PATH`.

*If you want to build ABINIT system-wide, you will need to build as the
super-user. By default, entering `root` does not have the proper environment
loaded. Load it with*

```sh
module load intel-compilers/16.2.181 intel-mpi/5.1.3.181 intel-mkl/16.2.181
```

*When compiling as root, it is best to only compile from the master node.*

It is convenient to have the following variables:

```sh
CC="icc"
CXX="icpc"
FC="ifort"
F77="ifort"
```

Check these and set them if not.


### Step 1 - Prepare the source code

You can find the tarballs for ABINIT and many other programs in
`/opt/science/tarballs/`.

Extract the ABINIT source code.

```sh
tar -xvf /opt/science/tarballs/abinit-7.10.5.tar.gz
cd abinit-7.10.5
```

We will modify the configure script to match our new compilers. This will
suppress many useless notifications and allow the build system to correctly set
the MPI compilers.

```sh
sed -i -e 's/vec-report0/qopt-report=0/g' \
       -e 's/mpicc/mpiicc/g' \
       -e 's/mpicxx/mpiicpc/g' \
       -e 's/mpif90/mpiifort/g' configure
```

I recommend creating a new directory for building. This makes for easier clean
up and for testing different compile options.

```
mkdir -p build && cd build
```

Copy the appropriate `abinit-x.x.x.ac` file to this directory, and rename it to
`medusa.ac` or whatever the hostname is of the node your are in.

```sh
cp abinit-x.x.x.ac ${HOSTNAME}.ac
```


### Step 2 - Configure and build

Initiate the configuration process by running

```sh
../configure
```

The script will read from the `.ac` file and set up the build system
accordingly. Review the process carefully and check for any mistakes or
problems. If everything goes as planned, it should produce this summary:

```
Summary of important options:

  * C compiler      : intel version 16.0
  * Fortran compiler: intel version 16.0
  * architecture    : intel xeon (64 bits)

  * debugging       : basic
  * optimizations   : yes

  * OpenMP enabled  : no (collapse: ignored)
  * MPI    enabled  : yes
  * MPI-IO enabled  : yes
  * GPU    enabled  : no (flavor: none)

  * TRIO   flavor = none
  * TIMER  flavor = abinit (libs: ignored)
  * LINALG flavor = mkl (libs: auto-detected)
  * ALGO   flavor = none (libs: ignored)
  * FFT    flavor = fftw3-mkl (libs: auto-detected)
  * MATH   flavor = none (libs: ignored)
  * DFT    flavor = none
```

You are now ready to build the software. You should parallelize this process to
significantly reduce compile time. For instance,

```sh
make multi multi_nprocs=8
```

will use 8 processor cores. **For this level of optimization, compiling will
take roughly 35 minutes on 8 cores.**


### Step 3 - Testing

We must now test our new ABINIT binaries. Refer to the ABINIT
[documentation]
(http://www.abinit.org/doc/helpfiles/for-v8.0/install_notes/install.html#make_tests)
for more information on testing. In the same build directory, create a
temporary directory for running the tests.

```sh
mkdir -p temp && cd temp
```

You can use the `$ABINIT/tests/runtests.py` program to thoroughly test your new
ABINIT build. Start off with

```sh
../../tests/runtests.py fast
```

to run the `fast` test suite, which are the most basic functionality tests. You
can also run several tests simultaneously,

```sh
../../tests/runtests.py -j8 fast
```

which will run 8 tests at once. Some tests are for testing parallelization, and
are designed to run with MPI threads. For instance,

```sh
../../tests/runtests.py -n4 paral
```

will run the `paral` test suite using 4 MPI processes. Some tests may be skipped
depending on the number of processors chosen, so you can try a few different
values. To run the entire battery of tests, run the `runtests.py` script without
specifying any specific test suite. I suggest something like

```sh
../../tests/runtests.py -j3 -n4
```

that will run the entire test suite, 3 tests at a time using up to 4 MPI
processes. This means that there will be between 3 and 12 processes running at a
given time. This will run over 600 tests. Many tests will be skipped since we
did not build all ABINIT features. It is also normal that a few tests will fail,
especially in the `v7` test suite that contains tests some experimental
connectors. Overall, your test failure rate should be less than 2%. The script
will output the test summary that you can review in the `Test_suite` directory.
You can view `suite_report.html` in your web browser for a very comprehensive
report on the tests.


### Step 4 - Installation and use

**Once you are satisfied with the testing, install ABINIT with `make install`.**

You can run ABINIT in the standard fashion for MPI binaries. For instance,

```
mpiexec.hydra -np 192 -hosts fat1,fat2,fat3 ~/bin/abinit < some.files
```

will run across nodes `fat1`, `fat2`, and `fat3` using 192 MPI processes (64
each). **It is no longer necessary to manually open and close an MPI ring when
initiating MPI parallelization with `mpiexec.hydra`.** See `I_MPI_FABRICS` if
you wish to select the
[network interface](https://software.intel.com/en-us/node/535584).

About optimization
-------------------------------------------------------

In general, compiler flags for optimizing on Medusa will look like

```sh
-axCORE-AVX2,SSE4.2 -ip -static-intel -fp-model precise -fp-model source -fma
```

These options enable the binaries to take advantage of the special processor
instructions available for the `hexa` (SSE4.2) and `fat` (CORE-AVX2) nodes (the
`medusa` node is also CORE-AVX2 enabled). There are downsides to this; mainly:
build times are 3 to 4 times longer than standard optimization, and the produced
binaries will NOT function on processors that do not have these instructions,
and will not likely work on any non-Intel processors. Obviously, these downsides
are negligible compared to the tremendous gains in speed and precision that can
be achieved with this level of optimization. However, please refer to the
[official](https://software.intel.com/en-us/article
s/performance-tools-for-software-developers-intel-compiler-options-for-sse-gener
ation-and-processor-specific-optimizations) [documentation](https://software.int
el.com/en-us/articles/performance-tools-for-software-developers-sse-generation-a
nd-processor-specific-optimizations-continue) for details about these options.

That being said, sometimes the software will not compile properly with these
options, so you should always check these flags and rigorously test after each
build. While troubleshooting and testing, I recommend a safer level of
optimization such as `-O2`.
