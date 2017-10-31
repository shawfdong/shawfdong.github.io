---
layout: post
title: Running PADDI on the Stampede2 Supercomputer
tags: [HPC]
---

In this post, we describe how to compile and run [Dr. Stephan Stellmach](http://earth.uni-muenster.de/~stellma/Site/Home.html)'s PADDI code on the [Stampede2](https://www.tacc.utexas.edu/systems/stampede2) supercomputer.<!-- more --> Phase I of Stampede2 rollout features 4,200 Knights Landing (KNL) nodes, the second generation of processors based on Intel's Many Integrated Core (MIC) architecture.

I copied the tarball for PADDI `PADDI_10.1_dist.tgz` from Hyades to Stampede2. The tarball contains generic x86-64 libraries and executable that have been compiled with Intel MPI and Intel Compilers. Since KNL processors offer binary compatibility with Intel Xeon processors, legacy x86-64 binaries can run on KNL nodes without recompilation. However, those binaries won't take advantage of KNL's unique features (such as AVX 512), and therefore won't run at optimal speed on KNL nodes. We'll recompile PADDI for the KNL microarchitecture.

Unpack the tarball in the home directory on Stampede2:
{% highlight shell %}
$ tar xvfz PADDI_10.1_dist.tgz
{% endhighlight %}

* Table of Contents
{:toc}

## Intel Compiler Flags
To build specifically for KNL microarchitecture using default Intel compilers, explicitly add the compiler flag `-xMIC-AVX512`. Or you can use the flags `-xCORE-AVX2 -axMIC-AVX512` to build a fat binary that contains optimized code for both Broadwell microarchitecture (the login nodes) and KNL (the Phase I compute nodes). In this post, we'll use `-xCORE-AVX2 -axMIC-AVX512`.

We note in passing that the default optimization level for Intel compilers is `-O2`.

## Dependencies
Let's first clean the house:
1. Delete all files in `stuff_needed/bin`
2. Delete all files in `stuff_needed/lib`
3. Delete all files except `jcmagic.h` in `stuff_needed/include`

According to the `README` file, PADDI depends upon the following libraries:
* FFTW3
* Parallel NetCDF
* a library called **jutils** (written by Joerg Schmalzl) used to save the data in compressed form

TACC provides optimized builds for *FFTW3* & *Parallel NetCDF* on Stampede2 and we'll simply use them. We'll only need to recompile *jutils*.

Note that TACC uses [LMOD](https://www.tacc.utexas.edu/research-development/tacc-projects/lmod) to manage software on Stampede2. LMOD is similar to the module utility deployed on Hyades and NERSC supercomputers. You can list currently loaded modules with:
{% highlight shell %}
$ module list

Currently Loaded Modules:
  1) intel/17.0.4   3) git/2.9.0       5) python/2.7.13   7) TACC
  2) impi/17.0.3    4) autotools/1.1   6) xalt/1.7.7
{% endhighlight %}

### FFTW3
To see what FFTW packages are available:
{% highlight shell %}
$ module avail fftw

-------------------- /opt/apps/intel17/impi17_0/modulefiles --------------------
   fftw2/2.1.5    fftw3/3.3.6
{% endhighlight %}

**fftw3** is what we need. To learn more about it:
{% highlight shell %}
$ module show fftw3
{% endhighlight %}

To load the *fftw3* module (note we only need to load the module when we compile PADDI, we *don't* need to load it when running PADDI):
{% highlight shell %}
$ module load fftw3
{% endhighlight %}

### Parallel netCDF (PnetCDF)
To see what NetCDF packages are avaiable:
{% highlight shell %}
$ module avail netcdf

-------------------- /opt/apps/intel17/impi17_0/modulefiles --------------------
   parallel-netcdf/4.3.3.1    pnetcdf/1.8.1

------------------------ /opt/apps/intel17/modulefiles -------------------------
   netcdf/4.3.3.1
{% endhighlight %}

The choices can be confusing, so a brief explanation is in order:
1. **parallel-netcdf** is parallel version of NetCDF4 based upon parallel hdf5; and it is *not* what we need
2. **pnetcdf** is parallel netcdf (PnetCDF) that supports netcdf in the classic formats (CDF-1 and CDF-2); and it is what we need
3. **netcdf** is the serial version of NetCDF4

To learn more about the *pnetcdf* module:
{% highlight shell %}
$ module show pnetcdf
{% endhighlight %}

To load the *pnetcdf* module (note we only need to load the module when we compile PADDI, we *don't* need to load it when running PADDI):
{% highlight shell %}
$ module load pnetcdf
{% endhighlight %}

### jutils
We'll recompile *jutils*. Go the source directory:
{% highlight shell %}
$ cd stuff_needed/lib_src/jutils
{% endhighlight %}

Clean up the old build:
{% highlight shell %}
$ make clean
{% endhighlight %}

Modify `Makefile` so that it will have the following contents (note that we simply add the `-xCORE-AVX2 -axMIC-AVX512` compiler flags):
{% highlight make %}
FC            = ifort
CFLAGS        = -O2 -xCORE-AVX2 -axMIC-AVX512 -Ijpeg12 -D_GNU_SOURCE  -DUSE12B -I.
FFLAGS        = -xCORE-AVX2 -axMIC-AVX512

DEST          = ${HOME}/bin

EXTHDRS       = /usr/include/getopt.h \
                /usr/include/sgidefs.h \
                /usr/include/stdio.h \
                /usr/include/stdlib.h \
                /usr/include/string.h

HDRS          = inout.h \
                jcmagic.h

LDFLAGS       =

LIBS          =

CC            = icc

LINKER        = $(CC)

MAKEFILE      = Makefile

OBJS          = icback_.o \
                icclose_.o \
                icopen_.o \
                icddopen_.o \
                icread_.o \
                icrinfo_.o \
                icskip_.o \
                icwinfo_.o \
                icdwinfo_.o \
                icddwinfo_.o \
                icwrite_.o \
                icdwrite_.o \
                icddwrite_.o \
                icend_.o \
                iread.o \
                jcback_.o \
                jcclose_.o \
                jcopen_.o \
                jcread_.o \
                jctype_.o \
                jcrinfo_.o \
                jcrinfo2_.o \
                jcskip_.o \
                jcwinfo_.o \
                jcwinfo2_.o \
                jcwrite_.o \
                jcend_.o \
                lread.o \
                rread.o \
                divisions.o \
#               jcclass.o

PRINT         = pr

PROGRAM       = a.out

SRCS          = icback_.c \
                icclose_.c \
                icopen_.c \
                icddopen_.c \
                icread_.c \
                icrinfo_.c \
                icskip_.c \
                icwinfo_.c \
                icdwinfo_.c \
                icddwinfo_.c \
                icwrite_.c \
                icdwrite_.c \
                icddwrite_.c \
                icend_.c \
                iread.f \
                jcback_.c \
                jcclose_.c \
                jcopen_.c \
                jcread_.c \
                jctype_.c \
                jcrinfo_.c \
                jcrinfo2_.c \
                jcskip_.c \
                jcwinfo_.c \
                jcwinfo2_.c \
                jcwrite_.c \
                jcend_.c \
                lread.f \
                rread.f \
                divisions.c \
#               jcclass.cpp

.F.o:
        $(FC)  -c -O2 $*.F


all:            libjc.a libjpeg

$(PROGRAM):     $(OBJS) $(LIBS) jpeg12/libjpeg.a
                $(LINKER) $(LDFLAGS) $(OBJS) $(LIBS) -o $(PROGRAM)

libjpeg:;       cd jpeg12; configure CC='icc -xCORE-AVX2 -axMIC-AVX512 -D_GNU_SOURCE  '; make
# libjpeg:;     cd jpeg12;configure CC='icc -D_GNU_SOURCE  -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE -D_LARGEFILE64_SOURCE'; make

libjc.a:         $(OBJS) $(LIBS)
                ar rv libjc.a $(OBJS)

c2j:             libjc.a c2j.o libjc.a jpeg12/libjpeg.a
                 ifc -o c2j  c2j.o libjc.a  jpeg12/libjpeg.a

j2c:             libjc.a j2c.o libjc.a jpeg12/libjpeg.a
                 ifc -o j2c  j2c.o libjc.a  jpeg12/libjpeg.a

greg2j: read_greg.o greg2j.o libjc.a jpeg12/libjpeg.a
                $(FC) $(FFLAGS) -o greg2j greg2j.o read_greg.o  libjc.a jpeg12/libjpeg.a

j2greg: write_greg.o j2greg.o
                $(FC) $(FFLAGS) -o j2greg j2greg.o write_greg.o  libjc.a jpeg12/libjpeg.a

read_greg.o:    read_greg.F
                $(FC) -c $(FFLAGS) read_greg.F

fchange:        fchange.o       fchange_cb.o    fchange_main.o
                $(CC) -o fchange fchange.o   fchange_cb.o  fchange_main.o -lforms -lXpm -lX11 -lm

clean:;         rm -f $(OBJS) *.bak *~ libjc.a; cd jpeg12; make clean

depend:;        mkmf -f $(MAKEFILE) PROGRAM=$(PROGRAM) DEST=$(DEST)

tar:;           cd jpeg12; make clean
                tar cvf jutils.tar Makefile *.c *.cpp *.h *.f *.F jpeg12 man converter/*.[c,h,f,F]
                gzip -fv jutils.tar

install:        $(PROGRAM)
                install -s $(PROGRAM) $(DEST)

print:;         $(PRINT) $(HDRS) $(SRCS)

program:        $(PROGRAM)

tags:           $(HDRS) $(SRCS); ctags $(HDRS) $(SRCS)

update:         $(DEST)/$(PROGRAM)

$(DEST)/$(PROGRAM): $(SRCS) $(LIBS) $(HDRS) $(EXTHDRS)
                @make -f $(MAKEFILE) DEST=$(DEST) install
###
{% endhighlight %}

Copy the newly built libraries:
{% highlight shell %}
$ cp libjc.a ../../lib/
$ cp jpeg12/libjpeg.a ../../lib/
{% endhighlight %}

## Compiling PADDI
Go to source directory for PADDI:
{% highlight shell %}
$ cd ../../../main/src
{% endhighlight %}

Clean up the old build:
{% highlight shell %}
$ make totalclean
{% endhighlight %}

Modify `Makefile` so that the first few lines will look as follows (the rest being the same as original):
{% highlight make %}
DEFS          = -DDOUBLE_PRECISION -DMPI_MODULE -DAB_BDF3 -DTEMPERATURE_FIELD -DCHEMICA
L_FIELD

FC            = mpiifort

F90           = $(FC)

FFLAGS        = $(DEFS) -xCORE-AVX2 -axMIC-AVX512

F90FLAGS      = -I../../stuff_needed/include -I${TACC_PNETCDF_INC} -I${TACC_FFTW3_INC}
-cpp $(DEFS)

LD            = $(FC)

LDFLAGS       =

LIBS          =

ADDLIBS       = -Wl,-rpath,${TACC_FFTW3_LIB} -L${TACC_FFTW3_LIB} -lfftw3f_mpi -lfftw3 -
L${TACC_PNETCDF_LIB} -lpnetcdf -L../../stuff_needed/lib -ljc -ljpeg -lirc
{% endhighlight %}

Recompile PADDI
{% highlight shell %}
$ make
{% endhighlight %}
It succeeded without a hitch!

## Running PADDI
Copy the executable (in this case, `double_diff_double_3D`) and other requisite files to your $SCRATCH directory.

PADDI is a pure MPI code. So we can write a SLURM job script based upon [the sample MPI job script in the user guide](https://portal.tacc.utexas.edu/user-guides/stampede2#submitting-batch-jobs-with-sbatch):
{% highlight shell %}
#!/bin/bash
#SBATCH -J DDtest           # Job name
#SBATCH -o DDtest.o%j       # Name of stdout output file
#SBATCH -e DDtest.e%j       # Name of stderr error file
#SBATCH -p normal          # Queue (partition) name
#SBATCH -N 4               # Total # of nodes
#SBATCH -n 256             # Total # of mpi tasks
#SBATCH -t 01:30:00        # Run time (hh:mm:ss)
#SBATCH --mail-user=pgaraud@soe.ucsc.edu
#SBATCH --mail-type=all    # Send email at begin and end of job

ibrun ./double_diff_double_3D <sample_parameter_file
{% endhighlight %}
Here we request 4 KNL nodes, and we run 64 MPI tasks per node (256 MPI tasks in total). There are 68 cores per node, so you should not run more than 68 MPI tasks per node. You might, however, want to run few tasks per node.

Use **sbatch** to submit your job.
