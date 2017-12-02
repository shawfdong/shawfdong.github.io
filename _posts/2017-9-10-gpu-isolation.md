---
layout: post
title: GPU Isolation
tags: [Docker, CUDA, GPU]
---

There are 4 [GeForce GTX 1080 Ti](https://www.nvidia.com/en-us/geforce/products/10series/geforce-gtx-1080-ti/) in the 4-GPU workstation [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}):<!-- more -->
{% highlight shell_session %}
# ls -l /dev/nvidia*
crw-rw-rw- 1 root root 195,   0 Sep  8 17:14 /dev/nvidia0
crw-rw-rw- 1 root root 195,   1 Sep  8 17:14 /dev/nvidia1
crw-rw-rw- 1 root root 195,   2 Sep  8 17:14 /dev/nvidia2
crw-rw-rw- 1 root root 195,   3 Sep  8 17:14 /dev/nvidia3
crw-rw-rw- 1 root root 195, 255 Sep  8 17:14 /dev/nvidiactl
crw-rw-rw- 1 root root 243,   0 Sep  8 17:14 /dev/nvidia-uvm
crw-rw-rw- 1 root root 243,   1 Sep  8 17:14 /dev/nvidia-uvm-tools

# lspci | grep -i nvidia
02:00.0 VGA compatible controller: NVIDIA Corporation Device 1b06 (rev a1)
03:00.0 VGA compatible controller: NVIDIA Corporation Device 1b06 (rev a1)
83:00.0 VGA compatible controller: NVIDIA Corporation Device 1b06 (rev a1)
84:00.0 VGA compatible controller: NVIDIA Corporation Device 1b06 (rev a1)
{% endhighlight %}

A caveat of running TensorFlow on a multi-GPU system such as Hydra is that by default, a TensorFlow session will allocate all GPU memory on all GPUs, even though you only use a single GPU! A better usage pattern is to launch multiple jobs in parallel, each one [using a subset of the available GPUs](https://github.com/NVIDIA/nvidia-docker/wiki/GPU-isolation).

Before you start any TensorFlow session, you should first run `nvidia-smi` to see which GPUs are being utilized, then select an idle GPU and target it with the environment variable [CUDA_VISIBLE_DEVICES](http://acceleware.com/blog/cudavisibledevices-masking-gpus).

For example, to select GPU 2 (*/dev/nvidia2*) to run your TensorFlow session, first set the environment variable:
{% highlight shell_session %}
export CUDA_VISIBLE_DEVICES=2
{% endhighlight %}
then launch Python to run TensorFlow.

Alternatively you can [set the environment variables within Python](https://stackoverflow.com/questions/37893755/tensorflow-set-cuda-visible-devices-within-jupyter), before you *import tensorflow*. For example: [CUDA_VISIBLE_DEVICES](http://acceleware.com/blog/cudavisibledevices-masking-gpus). For example:
{% highlight shell_session %}
# python
Python 3.6.2 |Anaconda custom (64-bit)| (default, Jul 20 2017, 13:51:32)
[GCC 4.4.7 20120313 (Red Hat 4.4.7-1)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> os.environ["CUDA_DEVICE_ORDER"]="PCI_BUS_ID"   # see issue #152
>>> os.environ["CUDA_VISIBLE_DEVICES"]="2"
>>> os.environ['TF_CPP_MIN_LOG_LEVEL']='2'
>>> import tensorflow as tf
>>> hello = tf.constant('Hello, TensorFlow!')
>>> sess = tf.Session()
>>> from tensorflow.python.client import device_lib
>>> print(device_lib.list_local_devices())
[name: "/cpu:0"
device_type: "CPU"
memory_limit: 268435456
locality {
}
incarnation: 17032249930653661893
, name: "/gpu:0"
device_type: "GPU"
memory_limit: 235274240
locality {
  bus_id: 2
}
incarnation: 9956660329545292162
physical_device_desc: "device: 0, name: GeForce GTX 1080 Ti, pci bus id: 0000:83:00.0"
]
{% endhighlight %}

<p class="note"> Here we also use the environmental variable <a href="https://stackoverflow.com/questions/35911252/disable-tensorflow-debugging-information">TF_CPP_MIN_LOG_LEVEL</a> to filter TensorFlow logs. It defaults to 0 (all logs shown), but can be set to 1 to filter out <em>INFO</em> logs, 2 to additionally filter out <em>WARNING</em> logs, and 3 to additionally filter out <em>ERROR</em> logs.</p>

If you use [nvidia-docker]({{ site.baseurl }}{% post_url 2017-9-9-docker-on-hydra %}#nvidia-docker), you can use the environment variable `NV_GPU` to isolate GPU. For example:
{% highlight shell_session %}
# NV_GPU=2 nvidia-docker run --rm nvidia/cuda nvidia-smi
Sun Sep 10 20:35:35 2017
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.66                 Driver Version: 384.66                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 108...  Off  | 00000000:83:00.0 Off |                  N/A |
| 23%   21C    P8    16W / 250W |  10623MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID  Type  Process name                               Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
{% endhighlight %}
