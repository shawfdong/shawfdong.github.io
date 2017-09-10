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
