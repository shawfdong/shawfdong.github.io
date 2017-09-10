---
layout: post
title: Docker on Hydra
tags: [Docker, CUDA]
---

In this post, we document how we installed [Docker](https://docs.docker.com/) on the 4-GPU workstation [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}).<!-- more -->

## Docker CE
We'll install [Docker CE (Community Edition)](https://docs.docker.com/engine/installation/linux/docker-ce/centos/) on Hydra, which runs CentOS 7.
1. Install required packages: `yum-utils` (providing the `yum-config-manager` utility), and `device-mapper-persistent-data` & `lvm2` (required by the `devicemapper` storage driver)
```shell
# yum install -y yum-utils device-mapper-persistent-data lvm2
```
2. Set up the *stable* repository for Docker CE:
```shell
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
3. Update the *yum* package index:
```shell
# yum makecache fast
```
4. Install the latest version of *Docker CE*:
```shell
# yum install docker-ce
```
5. Enable and start *Docker*:
```shell
# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
# systemctl start docker
```
6. Verify that *docker* is installed correctly:
```shell
# docker run --rm hello-world
```
We'll [allow non-privileged users to run Docker commands](https://docs.docker.com/engine/installation/linux/linux-postinstall/#manage-docker-as-a-non-root-user) later.

## nvidia-docker
[nvidia-docker](https://devblogs.nvidia.com/parallelforall/nvidia-docker-gpu-server-application-deployment-made-easy/) is essentially a wrapper around the *docker* command that transparently provisions a container with the necessary components to execute code on the GPU. It provides the two critical components needed for portable GPU-based containers:
* driver-agnostic CUDA images;
* a Docker command line wrapper that mounts the user mode components of the driver and the GPUs (character devices) into the container at launch.

1. [Install](https://github.com/NVIDIA/nvidia-docker/wiki) `nvidia-docker` & `nvidia-docker-plugin`:
```shell
# yum install https://github.com/NVIDIA/nvidia-docker/releases/download/v1.0.1/nvidia-docker-1.0.1-1.x86_64.rpm
```
2. Enable and start *nvidia-docker*:
```shell
# systemctl enable nvidia-docker
Created symlink from /etc/systemd/system/multi-user.target.wants/nvidia-docker.service to /usr/lib/systemd/system/nvidia-docker.service.
# systemctl start nvidia-docker
```
3. Verify that *nvidia-docker* is installed correctly:
```shell
# nvidia-docker run --rm nvidia/cuda nvidia-smi
Sun Sep 10 05:56:21 2017
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.66                 Driver Version: 384.66                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 108...  Off  | 00000000:02:00.0 Off |                  N/A |
| 23%   23C    P8     8W / 250W |     10MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  GeForce GTX 108...  Off  | 00000000:03:00.0 Off |                  N/A |
| 23%   27C    P8    10W / 250W |     10MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   2  GeForce GTX 108...  Off  | 00000000:83:00.0 Off |                  N/A |
| 23%   17C    P8     8W / 250W |     10MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   3  GeForce GTX 108...  Off  | 00000000:84:00.0 Off |                  N/A |
| 23%   21C    P8     9W / 250W |     10MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
```

## GPU isolation
There are 4 [GeForce GTX 1080 Ti](https://www.nvidia.com/en-us/geforce/products/10series/geforce-gtx-1080-ti/) in Hydra:
```shell
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
```

On a multi-GPUs machine, a common use case is to launch multiple jobs in parallel, each one [using a subset of the available GPUs](https://github.com/NVIDIA/nvidia-docker/wiki/GPU-isolation). There are a few solutions:

1) Using the the environment variable [CUDA_VISIBLE_DEVICES](http://acceleware.com/blog/cudavisibledevices-masking-gpus). For example, TensorFlow by default allocate all GPUs in a computer. To [allocate a single GPU](https://stackoverflow.com/questions/37893755/tensorflow-set-cuda-visible-devices-within-jupyter), e.g. */dev/nvidia2*, for a TensorFlow session:
```shell
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
```
<p class="note"> Here we also use the environmental variable <a href="https://stackoverflow.com/questions/35911252/disable-tensorflow-debugging-information">TF_CPP_MIN_LOG_LEVEL</a> to filter TensorFlow logs. It defaults to 0 (all logs shown), but can be set to 1 to filter out <em>INFO</em> logs, 2 to additionally filter out <em>WARNING</em> logs, and 3 to additionally filter out <em>ERROR</em> logs.</p>

2) using the environment variable `NV_GPU` for *nvidia-docker*:
```shell
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
```
