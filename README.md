Building Julia on knot.cnsi.ucsb.edu
====================================

[Knot](http://csc.cnsi.ucsb.edu/clusters/knot) is a cluster on the campus of UC Santa Barbara, located at the California NanoSystems Institute.

[Julia](http://julialang.org/) is a pretty sweet programming language.

This brief guide is designed to help users get Julia code running on knot.

Build instructions
------------------

It's best to **compile on the head node**, since it has network access, thus allowing Julia to fetch its dependencies automatically during a build.  Also, the most recent version of icc is currently only installed there.

I am personally using custom (more recent) compiles of gcc, Python, etc, so if anything below does not work flawlessly, this may be the issue.

We will plan to build using icc/mkl (see below for a note about compiling with gcc and OpenBLAS).  The first step is to import some environment variables:

    $ source /opt/intel/composer_xe_2015/bin/compilervars.sh intel64

We continue by cloning the source code and checking out the tag of the latest release (at the time of writing, `v0.3.9`).  Before this, you will want to get into a parent directory where you'd like the Julia installation to live.  Once you build it, you cannot change this directory without rebuilding (or so it seems).  Also, it is recommended to start from a fresh clone of the source repository.

    $ mkdir -p ~/julia
    $ cd ~/julia
    $ git clone --depth 50 https://github.com/JuliaLang/julia.git v0.3.9 -b v0.3.9
    $ cd v0.3.9
    $ git checkout v0.3.9

(Actually, the last step is redundant, but the `git-1.7.1` on knot is old enough that the `git clone` line will say `warning: Remote branch v0.3.9 not found in upstream origin, using HEAD instead`, so it's necessary to checkout the tag explicitly.  I'm assuming this is because `v0.3.9` is a tag, not a branch, but the same command works on later versions of `git`.)

On knot, not every CPU has the same instruction set, and this can cause the error "Target architecture mismatch" because Julia by default builds code that will only work on the architecture on which it was compiled.  On knot, the head node contains Westmere processors, but the big memory nodes contain X7550 chips, which are based on the older Nehalem microarchitecture.  Some of the newer nodes even contain Sandy Bridge processors.  To deal with this variation, we compile all code to target the lowest common denominator by specifying `JULIA_CPU_TARGET = nehalem` in `Make.user`.  For a full list of architecture options, see https://gcc.gnu.org/onlinedocs/gcc-4.9.2/gcc/i386-and-x86-64-Options.html#i386-and-x86-64-Options

Here's the full `Make.user` file which we put in the `v0.3.9` directory:

    JULIA_CPU_TARGET = nehalem

    USEICC = 1
    USEIFC = 1
    USE_INTEL_MKL = 1
    USE_INTEL_MKL_FFT = 1
    USE_INTEL_LIBM = 1

The last five lines are from Julia's [README](https://github.com/JuliaLang/julia#intel-compilers-and-math-kernel-library-mkl).

Now we are ready to build.

    $ make -j2

Once the build succeeds, you should be able to run Julia in the current directory:

    $ ./julia

At this point it's a smart move to run all tests:

    $ MKL_NUM_THREADS=4 make testall1

Assuming this works, edit `.bashrc` to pull in the path containing the binary:

    $ echo export PATH=`pwd`/usr/bin:\$PATH >> ~/.bashrc

It is also probably a good idea to have julia default to using a single core (unless a job explicitly asks for more):

    $ echo export MKL_NUM_THREADS=1 >> ~/.bashrc

Re-login, and run julia again.

    $ julia

Parallel processing
-------------------

Julia's parallel processing features can be used quite easily on knot.

    $ qsub -I -l nodes=4:ppn=1
    $ julia -q --machinefile $PBS_NODEFILE
	julia> @everywhere println("Hello, world!")
            From worker 5:	Hello, world!
            From worker 2:	Hello, world!
            From worker 4:	Hello, world!
            From worker 3:	Hello, world!

Parallel processing can be done just as easily from a non-interactive job as well.

PyCall
------

If you compile a custom version of Python to use on knot, it will need to be compiled as a shared library in order to work with Julia using [PyCall.jl](https://github.com/stevengj/PyCall.jl).  To do this, pass the `--enable-shared` flag to `./configure` when building Python.

Using gcc and OpenBLAS
----------------------

Instead of using icc/mkl, it is possible to use gcc and OpenBLAS (thus running an entirely Free Software build).  However, this combination of knot causes an intermittent segfault on exit unless `OPENBLAS_NUM_THREADS=1`.  Even a julia binary compiled on a different machine with the same version on Red Hat seems to have this issue on knot (but not on the other machine).  I have not been able to get to the bottom of this; it seems to be some strange interaction with the C library or something.

However, if you do wish to compile this way, put the single line `MARCH = nehalem` in `Make.user`.  Use `OPENBLAS_NUM_THREADS=1` at runtime (which can be set in `~/.bashrc`) to prevent the intermittent segfault on exit.

Acknowledgements
----------------

Thanks to Darwin Darakananda and the [SoCal Julia Meetup group](http://www.meetup.com/Southern-California-Julia-Users/) for helping me better understand the non-uniform architecture issue.
