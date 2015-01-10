Building Julia on knot.cnsi.ucsb.edu
====================================

[Knot](http://csc.cnsi.ucsb.edu/clusters/knot) is a cluster on the campus of UC Santa Barbara, located at the California NanoSystems Institute.

[Julia](http://julialang.org/) is a pretty sweet programming language.

This brief guide is designed to help users get Julia code running on knot.

TL;DR
-----

Put `MARCH=nehalem` in `Make.user`.  Use `OPENBLAS_NUM_THREADS=1` at runtime to prevent an intermittent segfault on exit.

Build instructions
------------------

It's best to compile on the head node, since it has network access, thus allowing Julia to fetch its dependencies automatically during a build.

I am personally using custom (more recent) compiles of gcc, Python, etc, so if anything below does not work flawlessly, this may be the issue.

Anyway, we start by cloning the source code and checking out the tag of the latest release (at the time of writing, `v0.3.5`).  Before this, you will want to get into a parent directory where you'd like the Julia installation to live.  Once you build it, you cannot change this directory without rebuilding (or so it seems).

    $ mkdir -p ~/julia
    $ cd ~/julia
    $ git clone --depth 50 https://github.com/JuliaLang/julia.git v0.3.5 -b v0.3.5
    $ cd v0.3.5

On knot, not every CPU has the same instruction set, and this can cause the error "Target architecture mismatch" because Julia by default builds code that will only work on the architecture on which it was compiled.  On knot, the head node contains Westmere processors, but the big memory nodes contain X7550 chips, which are based on the older Nehalem microarchitecture.  Some of the newer nodes even contain Sandy Bridge processors.  To deal with this variation, we compile all code to target the lowest common denominator:

    $ echo MARCH=nehalem > Make.user

For a full list of architecture options, see https://gcc.gnu.org/onlinedocs/gcc-4.9.2/gcc/i386-and-x86-64-Options.html#i386-and-x86-64-Options

Now we are ready to build.

    $ make

Once the build succeeds, you should be able to run Julia in the current directory:

    $ ./julia

Assuming this works, edit `.bashrc` to pull in the path containing the binary:

    $ echo export PATH=`pwd`/usr/bin:\$PATH >> ~/.bashrc

Re-login, and run julia again.

    $ julia

Open issues
-----------

The default build of Julia seems to intermittently segfault on exit on knot unless it is run with the environment variable `OPENBLAS_NUM_THREADS=1`.  This can be set in `.bashrc`.

It is also possible to build Julia using icc and MKL, but when Julia is built this way on knot, it fails any time it calls a subprocess.

	$ julia -e "run(\`true\`)" 
    Segmentation fault (core dumped)

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

Acknowledgements
----------------

Thanks to Darwin Darakananda and the [SoCal Julia Meetup group](http://www.meetup.com/Southern-California-Julia-Users/) for helping me better understand the non-uniform architecture issue.
