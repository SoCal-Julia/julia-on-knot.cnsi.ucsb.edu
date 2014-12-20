Building Julia on knot.cnsi.ucsb.edu
====================================

[Knot](http://csc.cnsi.ucsb.edu/clusters/knot) is a cluster on the campus of UC Santa Barbara, located at the California NanoSystems Institute.

[Julia](http://julialang.org/) is a pretty sweet programming language.

This brief guide is designed to help users get Julia code running on knot.

Build instructions
------------------

It's best to compile on the head node, since it has network access, thus allowing Julia to fetch its dependencies automatically during a build.

I am personally using custom (more recent) compiles of gcc, etc, so if anything below does not work flawlessly, this may be the issue.

Anyway, we start by cloning the source code and checking out the `release-0.3` branch.  Before this, you will want to get into a parent directory where you'd like the Julia installation to live.  Once you build it, you cannot change this directory without rebuilding.

    $ git clone https://github.com/JuliaLang/julia.git julia-release-0.3
    $ cd julia-release-0.3
    $ git checkout release-0.3

On knot, not every CPU has the same instruction set, and this can cause problems because Julia by default builds code that will only work on the architecture on which it was compiled.  On knot, the head node contains Westmere processors, but the big memory nodes contain X7550 chips, which are based on the older Nehalem microarchitecture.  To deal with this, we compile all code to target the lowest common denominator:

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

The default build of Julia seems to intermittently segfault on exit unless we set the environment variable `OPENBLAS_NUM_THREADS=1`.  This can be done in `.bashrc`.

It is also possible to build Julia using icc and MKL, but when Julia is built this way on knot, it fails any time it calls a subprocess.

	$ julia -e "run(\`true\`)" 
    Segmentation fault (core dumped)

Acknowledgements
----------------

Thanks to Darwin Darakananda and the SoCal Julia Meetup group for helping me better understand the non-uniform architecture issue.
