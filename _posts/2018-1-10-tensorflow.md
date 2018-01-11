---
layout: post
title: TensorFlow 1.4 on Hydra
tags: [GPU, CUDA, Linux, Machine Learning, Python]
---

In this post, we describe how we installed TensorFlow 1.4 on the 4-GPU workstation [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}).<!-- more --> Previously, we installed [TensorFlow 1.3 with Anaconda]({{ site.baseurl }}{% post_url 2017-12-1-anaconda-tensorflow %}) on Hydra. But because there are a lot of [major features and improvements introduced in release 1.4](https://github.com/tensorflow/tensorflow/releases), it is worthwhile to spend some effort to install the latest release of TensorFlow on the GPU box.

* Table of Contents
{:toc}

## Python 3.6 compatibility

The latest Anaconda3 version (5.0.1 as of this writing) includes Python 3.6 by default. As noted in [previous post]({{ site.baseurl }}{% post_url 2017-12-1-anaconda-tensorflow %}), if we attempt a ["native" pip](https://www.tensorflow.org/install/install_linux#InstallingNativePip) of the official TensorFlow release:
{% highlight shell_session %}
[root@hydra ~]# pip install tensorflow-gpu
Collecting tensorflow-gpu
  Downloading tensorflow_gpu-1.4.1-cp36-cp36m-manylinux1_x86_64.whl (170.3MB)
    100% |████████████████████████████████| 170.3MB 8.2kB/s
...
  Downloading tensorflow_tensorboard-0.4.0rc3-py3-none-any.whl (1.7MB)
    100% |████████████████████████████████| 1.7MB 863kB/s
...
Successfully installed bleach-1.5.0 enum34-1.1.6 html5lib-0.9999999 markdown-2.6.11 protobuf-3.5.1 tensorflow-gpu-1.4.1 tensorflow-tensorboard-0.4.0rc3
{% endhighlight %}
the installation will fail to pass the simple validation:
{% highlight shell_session %}
[root@hydra ~]# python
Python 3.6.3 |Anaconda, Inc.| (default, Oct 13 2017, 12:02:49)
[GCC 7.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
/opt/tensorflow14/lib/python3.6/importlib/_bootstrap.py:219: RuntimeWarning: compiletime version 3.5 of module 'tensorflow.python.framework.fast_tensor_util' does not match runtime version 3.6
  return f(*args, **kwds)
{% endhighlight %}
It looks like we need to downgrade to Python 3.5!

## Anaconda Python 3.5
There are [three ways to downgrade to Anaconda Python 3.5](https://docs.anaconda.com/anaconda/faq#how-do-i-get-anaconda-with-python-3-5):
* Install the latest version of Anaconda and then make a [Python 3.5 environment](https://conda.io/docs/py2or3.html)
* Install the latest version of Anaconda and run this command to install Python 3.5 in the root environment: `conda install python=3.5`
* Install the most recent Anaconda that included Python 3.5 by default, [Anaconda 4.2.0](https://repo.continuum.io/archive/)

We'll take the second approach.

1) Install a separate copy of Anaconda3 at `/opt/tensorflow14` so that it will be accessible by all users on Hydra:
{% highlight shell_session %}
# cd ~/Downloads
# ./Anaconda3-5.0.1-Linux-x86_64.sh
{% endhighlight %}

2) Create an [environment module]({{ site.baseurl }}{% post_url 2017-9-8-environment-modules %}), `python/tensorflow14`, for this Anaconda3 installation. Then load the module:
{% highlight shell_session %}
# module load python/tensorflow14
{% endhighlight %}

3) Install Python 3.5 in the root environment:
{% highlight shell_session %}
# conda install python=3.5

# which python
/opt/tensorflow14/bin/python
# python --version
Python 3.5.4 :: Anaconda custom (64-bit)
{% endhighlight %}

4) Perform a ["native" pip](https://www.tensorflow.org/install/install_linux#InstallingNativePip) install of the official TensorFlow release:
{% highlight shell_session %}
[root@hydra Downloads]# pip install tensorflow-gpu
Collecting tensorflow-gpu
  Downloading tensorflow_gpu-1.4.1-cp35-cp35m-manylinux1_x86_64.whl (170.1MB)
    100% |████████████████████████████████| 170.1MB 8.3kB/s
...
Successfully installed bleach-1.5.0 enum34-1.1.6 html5lib-0.9999999 markdown-2.6.11 protobuf-3.5.1 tensorflow-gpu-1.4.1 tensorflow-tensorboard-0.4.0rc3
{% endhighlight %}

## Weak-modules woes
Unfortunately, the installation still couldn't pass the simple validation test, albeit with a new error!
{% highlight shell_session %}
# python
Python 3.5.4 |Anaconda custom (64-bit)| (default, Nov 20 2017, 18:44:38)
[GCC 7.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> sess = tf.Session()
2018-01-10 11:32:30.927717: E tensorflow/stream_executor/cuda/cuda_driver.cc:406] failed call to cuInit: CUDA_ERROR_UNKNOWN
{% endhighlight %}

After some digging, I found something might be amiss with the Nvidia driver:
{% highlight shell_session %}
# modinfo nvidia
filename:       /lib/modules/3.10.0-693.11.6.el7.x86_64/weak-updates/nvidia.ko
modinfo: ERROR: could not get modinfo from 'nvidia': No such file or directory

# cd /lib/modules/3.10.0-693.11.6.el7.x86_64/weak-updates/
# ls -l nvidia*
total 0
lrwxrwxrwx 1 root root 58 Jan  4 09:45 nvidia-drm.ko -> /lib/modules/3.10.0-693.5.2.el7.x86_64/extra/nvidia-drm.ko
lrwxrwxrwx 1 root root 54 Jan  4 09:45 nvidia.ko -> /lib/modules/3.10.0-693.5.2.el7.x86_64/extra/nvidia.ko
lrwxrwxrwx 1 root root 62 Jan  4 09:45 nvidia-modeset.ko -> /lib/modules/3.10.0-693.5.2.el7.x86_64/extra/nvidia-modeset.ko
lrwxrwxrwx 1 root root 58 Jan  4 09:45 nvidia-uvm.ko -> /lib/modules/3.10.0-693.5.2.el7.x86_64/extra/nvidia-uvm.ko
{% endhighlight %}
However, the targets of those symbolic links were no longer existent on Hydra! Now I recall that in the first week of 2018 I upgraded all the packages on Hydra, then removed the old kernel packages after the upgrade. What I didn't realize then was that the simple command `yum erase kernel-3.10.0-693.5.2.el7.x86_64` would also erase the Nvidia kernel drivers that were required by the new kernel!

Once we knew what the problem was, it was straightforward to fix it.

1) Delete the dangling symbolic links:
{% highlight shell_session %}
# rm nvidia*
rm: remove symbolic link ‘nvidia-drm.ko’? y
rm: remove symbolic link ‘nvidia.ko’? y
rm: remove symbolic link ‘nvidia-modeset.ko’? y
rm: remove symbolic link ‘nvidia-uvm.ko’? y
{% endhighlight %}

2) Erase the *nvidia-kmod* package:
{% highlight shell_session %}
# yum erase -y nvidia-kmod
================================================================================
Removing:
 nvidia-kmod                   x86_64     1:387.26-2.el7        @cuda      23 M
Removing for dependencies:
 cuda                          x86_64     9.1.85-1              @cuda     0.0
 cuda-8-0                      x86_64     8.0.61-1              @cuda     0.0
 cuda-9-0                      x86_64     9.0.176-1             @cuda     0.0
 cuda-9-1                      x86_64     9.1.85-1              @cuda     0.0
 cuda-demo-suite-8-0           x86_64     8.0.61-1              @cuda      11 M
 cuda-demo-suite-9-0           x86_64     9.0.176-1             @cuda      11 M
 cuda-demo-suite-9-1           x86_64     9.1.85-1              @cuda      11 M
 cuda-drivers                  x86_64     387.26-1              @cuda     0.0
 cuda-runtime-8-0              x86_64     8.0.61-1              @cuda     0.0
 cuda-runtime-9-0              x86_64     9.0.176-1             @cuda     0.0
 cuda-runtime-9-1              x86_64     9.1.85-1              @cuda     0.0
 xorg-x11-drv-nvidia           x86_64     1:387.26-1.el7        @cuda      11 M
 xorg-x11-drv-nvidia-devel     x86_64     1:387.26-1.el7        @cuda     579 k
 xorg-x11-drv-nvidia-gl        x86_64     1:387.26-1.el7        @cuda      75 M
 xorg-x11-drv-nvidia-libs      x86_64     1:387.26-1.el7        @cuda      93 M
{% endhighlight %}

3) Reboot.

4) Reinstall CUDA:
{% highlight shell_session %}
# yum install -y cuda
{% endhighlight %}

5) Reboot one more time. Now the latest Nvidia driver is properly installed and loaded:
{% highlight shell_session %}
# modinfo nvidia
filename:       /lib/modules/3.10.0-693.11.6.el7.x86_64/extra/nvidia.ko
alias:          char-major-195-*
version:        387.26
supported:      external
license:        NVIDIA
rhelversion:    7.4
...
{% endhighlight %}

## Validation
Finally TensorFlow 1.4.1 appears to work as expected!
{% highlight shell_session %}
$ module load python/tensorflow14
$ python
Python 3.5.4 |Anaconda custom (64-bit)| (default, Nov 20 2017, 18:44:38)
[GCC 7.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
>>> b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
>>> c = tf.matmul(a, b)
>>> sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
2018-01-10 12:23:56.993368: I tensorflow/core/platform/cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: SSE4.1 SSE4.2 AVX AVX2 FMA
2018-01-10 12:24:02.603082: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 0 with properties:
name: GeForce GTX 1080 Ti major: 6 minor: 1 memoryClockRate(GHz): 1.582
pciBusID: 0000:02:00.0
totalMemory: 10.91GiB freeMemory: 10.75GiB
2018-01-10 12:24:02.995246: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 1 with properties:
name: GeForce GTX 1080 Ti major: 6 minor: 1 memoryClockRate(GHz): 1.582
pciBusID: 0000:03:00.0
totalMemory: 10.91GiB freeMemory: 10.75GiB
2018-01-10 12:24:03.369357: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 2 with properties:
name: GeForce GTX 1080 Ti major: 6 minor: 1 memoryClockRate(GHz): 1.582
pciBusID: 0000:83:00.0
totalMemory: 10.91GiB freeMemory: 10.75GiB
2018-01-10 12:24:03.717394: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 3 with properties:
name: GeForce GTX 1080 Ti major: 6 minor: 1 memoryClockRate(GHz): 1.582
pciBusID: 0000:84:00.0
totalMemory: 10.91GiB freeMemory: 10.75GiB
2018-01-10 12:24:03.723314: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1045] Device peer to peer matrix
2018-01-10 12:24:03.723923: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1051] DMA: 0 1 2 3
2018-01-10 12:24:03.723950: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1061] 0:   Y Y N N
2018-01-10 12:24:03.723969: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1061] 1:   Y Y N N
2018-01-10 12:24:03.723988: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1061] 2:   N N Y Y
2018-01-10 12:24:03.724007: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1061] 3:   N N Y Y
2018-01-10 12:24:03.724049: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:0) -> (device: 0, name: GeForce GTX 1080 Ti, pci bus id: 0000:02:00.0, compute capability: 6.1)
2018-01-10 12:24:03.724074: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:1) -> (device: 1, name: GeForce GTX 1080 Ti, pci bus id: 0000:03:00.0, compute capability: 6.1)
2018-01-10 12:24:03.724136: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:2) -> (device: 2, name: GeForce GTX 1080 Ti, pci bus id: 0000:83:00.0, compute capability: 6.1)
2018-01-10 12:24:03.724158: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:3) -> (device: 3, name: GeForce GTX 1080 Ti, pci bus id: 0000:84:00.0, compute capability: 6.1)
Device mapping:
/job:localhost/replica:0/task:0/device:GPU:0 -> device: 0, name: GeForce GTX 1080 Ti, pci bus id: 0000:02:00.0, compute capability: 6.1
/job:localhost/replica:0/task:0/device:GPU:1 -> device: 1, name: GeForce GTX 1080 Ti, pci bus id: 0000:03:00.0, compute capability: 6.1
/job:localhost/replica:0/task:0/device:GPU:2 -> device: 2, name: GeForce GTX 1080 Ti, pci bus id: 0000:83:00.0, compute capability: 6.1
/job:localhost/replica:0/task:0/device:GPU:3 -> device: 3, name: GeForce GTX 1080 Ti, pci bus id: 0000:84:00.0, compute capability: 6.1
2018-01-10 12:24:04.190943: I tensorflow/core/common_runtime/direct_session.cc:299] Device mapping:
/job:localhost/replica:0/task:0/device:GPU:0 -> device: 0, name: GeForce GTX 1080 Ti, pci bus id: 0000:02:00.0, compute capability: 6.1
/job:localhost/replica:0/task:0/device:GPU:1 -> device: 1, name: GeForce GTX 1080 Ti, pci bus id: 0000:03:00.0, compute capability: 6.1
/job:localhost/replica:0/task:0/device:GPU:2 -> device: 2, name: GeForce GTX 1080 Ti, pci bus id: 0000:83:00.0, compute capability: 6.1
/job:localhost/replica:0/task:0/device:GPU:3 -> device: 3, name: GeForce GTX 1080 Ti, pci bus id: 0000:84:00.0, compute capability: 6.1

>>> print(sess.run(c))
MatMul: (MatMul): /job:localhost/replica:0/task:0/device:GPU:0
2018-01-10 12:24:27.442923: I tensorflow/core/common_runtime/placer.cc:874] MatMul: (MatMul)/job:localhost/replica:0/task:0/device:GPU:0
b: (Const): /job:localhost/replica:0/task:0/device:GPU:0
2018-01-10 12:24:27.443079: I tensorflow/core/common_runtime/placer.cc:874] b: (Const)/job:localhost/replica:0/task:0/device:GPU:0
a: (Const): /job:localhost/replica:0/task:0/device:GPU:0
2018-01-10 12:24:27.443159: I tensorflow/core/common_runtime/placer.cc:874] a: (Const)/job:localhost/replica:0/task:0/device:GPU:0
[[ 22.  28.]
 [ 49.  64.]]
>>> sess.close()
>>> quit()
{% endhighlight %}
