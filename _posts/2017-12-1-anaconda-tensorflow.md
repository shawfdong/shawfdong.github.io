---
layout: post
title: Installing TensorFlow with Anaconda
tags: [GPU, CUDA, Linux, Machine Learning]
---

In this post, we describe how we installed TensorFlow with Anaconda on the 4-GPU workstation [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}).<!-- more -->

* Table of Contents
{:toc}

## Anaconda Python
[Anaconda](https://docs.anaconda.com/anaconda/) is a freemium open source distribution of Python. Anaconda provides high performance computing with:
* [Anaconda Accelerate](https://docs.anaconda.com/accelerate/index.html)
* [MKL Optimizations](https://docs.anaconda.com/mkl-optimizations/index.html)
* [IOPro](https://docs.anaconda.com/iopro/index.html)
* [NumbaPro](https://docs.anaconda.com/numbapro/index.html)

One should use Anaconda Python, rather than the stock Python that comes with your Linux distros, for any serious computation.

1) Download [the latest Anaconda3 for Linux Installer](https://www.continuum.io/downloads#linux) (v5.0.1 as of this writing) and install it:
{% highlight shell_session %}
# wget https://repo.continuum.io/archive/Anaconda3-5.0.1-Linux-x86_64.sh
# chmod +x Anaconda3-5.0.1-Linux-x86_64.sh
# ./Anaconda3-5.0.1-Linux-x86_64.sh
{% endhighlight %}
This will install Python 3.6. The default install location is `$HOME/anaconda3`, so any user can install a private copy. I chose to install it at system location `/opt/anaconda3` and thus made it available to all users.

2) Update Anaconda 3:
{% highlight shell_session %}
# conda update conda
# conda update anaconda
# conda update python
# conda update --all
{% endhighlight %}

3) Download [the latest Anaconda2 for Linux Installer](https://www.continuum.io/downloads#linux) (v5.0.1 as of this writing) and install it:
{% highlight shell_session %}
# wget https://repo.continuum.io/archive/Anaconda2-5.0.1-Linux-x86_64.sh
# chmod +x Anaconda2-5.0.1-Linux-x86_64.sh
# ./Anaconda2-5.0.1-Linux-x86_64.sh
{% endhighlight %}
This will install Python 2.7. The default install location is `$HOME/anaconda2`, so any user can install a private copy. I chose to install it at system location `/opt/anaconda2` and thus made it available to all users.

4) Update Anaconda 2:
{% highlight shell_session %}
# module swap python/anaconda3 python/anaconda2
# which python
/opt/anaconda2/bin/python

# conda update conda
# conda update anaconda
# conda update python
# conda update --all
{% endhighlight %}

**Note** here we use [module]({{ site.baseurl }}{% post_url 2017-9-8-environment-modules %}) to facilitate the usage of multiple Python distributions on Hydra.

One crucial reason that Anaconda Python provides much higher performance than the stock Python is that it uses the highly optimized Intel MKL for some of most popular numerical/scientific Python libraries, including NumPy, SciPy & Scikit-Learn:  
{% highlight shell_session %}
$ python
Python 3.6.3 |Anaconda custom (64-bit)| (default, Nov 20 2017, 20:41:42)
[GCC 7.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import numpy as np
>>> np.show_config()
blas_mkl_info:
    libraries = ['mkl_rt', 'pthread']
    library_dirs = ['/opt/anaconda3/lib']
    define_macros = [('SCIPY_MKL_H', None), ('HAVE_CBLAS', None)]
    include_dirs = ['/opt/anaconda3/include']
blas_opt_info:
    libraries = ['mkl_rt', 'pthread']
    library_dirs = ['/opt/anaconda3/lib']
    define_macros = [('SCIPY_MKL_H', None), ('HAVE_CBLAS', None)]
    include_dirs = ['/opt/anaconda3/include']
lapack_mkl_info:
    libraries = ['mkl_rt', 'pthread']
    library_dirs = ['/opt/anaconda3/lib']
    define_macros = [('SCIPY_MKL_H', None), ('HAVE_CBLAS', None)]
    include_dirs = ['/opt/anaconda3/include']
lapack_opt_info:
    libraries = ['mkl_rt', 'pthread']
    library_dirs = ['/opt/anaconda3/lib']
    define_macros = [('SCIPY_MKL_H', None), ('HAVE_CBLAS', None)]
    include_dirs = ['/opt/anaconda3/include']
{% endhighlight %}

## Incorrect way of installing TensorFlow with Anaconda
Before I show you the proper way of installing TensorFlow with Anaconda, I'd like to point out that there are a couple of deficiencies in the official TensorFlow documentation on [Installing with Anaconda](https://www.tensorflow.org/install/install_linux#InstallingAnaconda). Let's follow the instructions step-by-step.

1) We've already downloaded and installed Anaconda.

2) Create a conda environment named `tensorflow` to run Python 3.6 (as an unprivileged user):
{% highlight shell_session %}
$ module load python
$ which python
/opt/anaconda3/bin/python

$ conda create -n tensorflow python=3.6
{% endhighlight %}

3) Activate the conda environment:
{% highlight shell_session %}
$ source activate tensorflow
{% endhighlight %}

4) Install the latest TensorFlow release (1.4.0 as of this writing) inside the conda environment:
{% highlight shell_session %}
(tensorflow)$ pip install --ignore-installed --upgrade https://storage.googleapis.com/tensorflow/linux/gpu/tensorflow_gpu-1.4.0-cp36-cp36m-linux_x86_64.whl
...
Successfully installed bleach-1.5.0 enum34-1.1.6 html5lib-0.9999999 markdown-2.6.9 numpy-1.13.3 protobuf-3.5.0.post1 setuptools-38.2.3 six-1.11.0 tensorflow-gpu-1.4.0 tensorflow-tensorboard-0.4.0rc3 werkzeug-0.12.2 wheel-0.30.0
{% endhighlight %}

However, when I tried to import the *tensorflow* module, I got an error:
{% highlight shell_session %}
(tensorflow)$ python
>>> import tensorflow as tf
/home/dong/.conda/envs/tensorflow/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: compiletime version 3.5 of module 'tensorflow.python.framework.fast_tensor_util' does not match runtime version 3.6
  return f(*args, **kwds)
{% endhighlight %}
Apparently, the module was compiled against the wrong Python version! Admittedly, it might work to install TensorFlow this way with earlier Python versions, such as 2.7 and 3.5; but not with 3.6!

In the above output we note that *numpy* was also installed in the conda environment, as a dependency of TensorFlow. However, this numpy module is not built with MKL, but rather with OpenBLAS!
{% highlight shell_session %}
(tensorflow)$ python
>>> import numpy as np
>>> np.show_config()
blas_mkl_info:
  NOT AVAILABLE
blis_info:
  NOT AVAILABLE
openblas_info:
    libraries = ['openblas', 'openblas']
    library_dirs = ['/usr/local/lib']
    language = c
    define_macros = [('HAVE_CBLAS', None)]
blas_opt_info:
    libraries = ['openblas', 'openblas']
    library_dirs = ['/usr/local/lib']
    language = c
    define_macros = [('HAVE_CBLAS', None)]
lapack_mkl_info:
  NOT AVAILABLE
openblas_lapack_info:
    libraries = ['openblas', 'openblas']
    library_dirs = ['/usr/local/lib']
    language = c
    define_macros = [('HAVE_CBLAS', None)]
lapack_opt_info:
    libraries = ['openblas', 'openblas']
    library_dirs = ['/usr/local/lib']
    language = c
    define_macros = [('HAVE_CBLAS', None)]
{% endhighlight %}

The OpenBLAS libraries are presumably located in `/usr/local/lib`. However, that directory is empty!

It's not worth our time to investigate further. Let's deactivate the conda environment:
{% highlight shell_session %}
(tensorflow) [dong@hydra ~]$ source deactivate
{% endhighlight %}

then remove the conda environment:
{% highlight shell_session %}
$ conda env remove -n tensorflow
{% endhighlight %}

## Proper way of installing TensorFlow with Anaconda
In fact, Conda provides TensorFlow packages in its default channel / repository! As of this writing, the latest TensorFlow version in the default channel is 1.3.0, which is slightly behind the latest official release of TensorFlow (1.4.0).

Installing TensorFlow is easy:
{% highlight shell_session %}
# conda install tensorflow-gpu
Fetching package metadata ...........
Solving package specifications: .

Package plan for installation in environment /opt/anaconda3:

The following NEW packages will be INSTALLED:

    backports.weakref:      1.0rc1-py36_0
    cudatoolkit:            8.0-3
    cudnn:                  6.0.21-cuda8.0_0
    libgcc:                 7.2.0-h69d50b8_2
    libprotobuf:            3.4.1-h5b8497f_0
    markdown:               2.6.9-py36_0
    protobuf:               3.4.1-py36h306e679_0
    tensorflow-gpu:         1.3.0-0
    tensorflow-gpu-base:    1.3.0-py36cuda8.0cudnn6.0_1
    tensorflow-tensorboard: 0.1.5-py36_0

The following packages will be DOWNGRADED:

    bleach:                 2.1.1-py36hd521086_0        --> 1.5.0-py36_0
    html5lib:               0.999999999-py36h2cfc398_0  --> 0.9999999-py36_0

{% endhighlight %}
**Note** Conda installed its own copy of CUDA 8.0 & cuDNN 6.0 in Anaconda, so it doesn't depend upon external CUDA & cuDNN libraries to function!

Similarly, we've also installed TensorFlow with Anaconda2:
{% highlight shell_session %}
# module swap python python/anaconda2
# which conda
/opt/anaconda2/bin/conda

# conda install tensorflow-gpu
{% endhighlight %}

## Installing Keras with Anaconda
At this point, it should be no surprise that Keras is also included in the default conda channel; so installing Keras is also a breeze.

Install Keras with Anaconda3:
{% highlight shell_session %}
# which conda
/opt/anaconda3/bin/conda

# conda install keras-gpu
{% endhighlight %}

Install Keras with Anaconda2:
{% highlight shell_session %}
# module swap python python/anaconda2
# which conda
/opt/anaconda2/bin/conda

# conda install keras-gpu
{% endhighlight %}
