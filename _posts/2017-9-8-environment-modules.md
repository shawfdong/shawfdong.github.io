---
layout: post
title: Environment Modules
tags: [Linux, CUDA, Python, HPC]
---

We set up [module](http://modules.sourceforge.net/) on [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}) to facilitate management and usage of different versions of a single software, such as multiple Python distributions, on the same machine.<!-- more --> Many supercomputing centers, including [NERSC](http://www.nersc.gov/users/software/nersc-user-environment/modules/), use the `module` utility to manage software.

## module
The module utility is available from CentOS `base` repo. So one can simply install it with:
```shell
# yum -y install environment-modules
```

## Python
So far we've installed 2 Python distributions, not counting the stock Python 2.7.5 that comes with CentOS7, on Hydra: *Anaconda3 4.4.0 for Python 3.6* & *Anaconda2 4.4.0 for Python 2.7*.

Create a directory `/etc/modulefiles/python` to hold modulefiles for Python distributions:
```shell
# mkdir /etc/modulefiles/python
```

Create a modulefile `/etc/modulefiles/python/anaconda3` for *Anaconda Python 3.6*:
```tcl
#%Module1.0#####################################################################
##
## anaconda3 modulefile
##
proc ModulesHelp { } {
        puts stderr "\tAdds Anaconda Python 3 to your PATH environment variable\n"
}

module-whatis   "adds Anaconda Python 3 to your PATH environment variable"

prepend-path    PATH    /opt/anaconda3/bin
conflict        python
```

Create a modulefile `/etc/modulefiles/python/anaconda2` for *Anaconda Python 2.7*:
```tcl
#%Module1.0#####################################################################
##
## anaconda3 modulefile
##
proc ModulesHelp { } {
        puts stderr "\tAdds Anaconda Python 2 to your PATH environment variable\n"
}

module-whatis   "adds Anaconda Python 2 to your PATH environment variable"

prepend-path    PATH    /opt/anaconda2/bin
conflict        python
```

Create a file `/etc/modulefiles/python/.version` to make `python\anaconda3` the default *python* module:
```tcl
#%Module
set ModulesVersion "anaconda3"
```

A few quick examples:
```shell
# which python
/usr/bin/python

# module load python
# which python
/opt/anaconda3/bin/python

# module load python/anaconda2
python/anaconda2(12):ERROR:150: Module 'python/anaconda2' conflicts with the currently loaded module(s) 'python/anaconda3'
python/anaconda2(12):ERROR:102: Tcl command execution failed: conflict  python

# module swap python python/anaconda2
# which python
/opt/anaconda2/bin/python
```

## CUDA
So far we've only installed *CUDA 8.0* on Hydra; but we'll install new versions when they become available.

Create a directory `/etc/modulefiles/cuda` to hold modulefiles for CUDA:
```shell
# mkdir /etc/modulefiles/cuda
```

Create a modulefile `/etc/modulefiles/cuda/8.0` for *CUDA 8.0*:
```tcl
#%Module1.0#####################################################################
##
## cuda 8.0 modulefile
##
proc ModulesHelp { } {
        global version

        puts stderr "\tSets up environment for CUDA $version\n"
}

module-whatis   "sets up environment for CUDA 8.0"

# for Tcl script use only
set     version 8.0
set     root    /usr/local/cuda-${version}

setenv  CUDA_HOME       $root

append-path     PATH    $root/bin
append-path     MANPATH $root/doc/man

conflict cuda
```

Create a file `/etc/modulefiles/cuda/.version` to make `cuda\8.0` the default *cuda* module:
```tcl
#%Module
set ModulesVersion "8.0"
```

## Loading modules into your default environment
To automatically load the default `python` & `cuda` modules to your environment whenever you log in, append the following 2 lines to `~/.bashrc` (assuming your shell is *bash*):
```shell
module load cuda
module load python
```
