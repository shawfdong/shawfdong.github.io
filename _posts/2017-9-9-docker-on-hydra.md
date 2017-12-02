---
layout: post
title: Docker on Hydra
tags: [Docker, CUDA]
---

In this post, we document how we installed [Docker](https://docs.docker.com/) on the 4-GPU workstation [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}).<!-- more -->

* Table of Contents
{:toc}

## Docker CE
We'll install [Docker CE (Community Edition)](https://docs.docker.com/engine/installation/linux/docker-ce/centos/) on Hydra, which runs CentOS 7.

1) Install required packages: `yum-utils` (providing the `yum-config-manager` utility), and `device-mapper-persistent-data` & `lvm2` (required by the `devicemapper` storage driver)
{% highlight shell_session %}
# yum install -y yum-utils device-mapper-persistent-data lvm2
{% endhighlight %}

2) Set up the *stable* repository for Docker CE:
{% highlight shell_session %}
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
{% endhighlight %}

3) Update the *yum* package index:
{% highlight shell_session %}
# yum makecache fast
{% endhighlight %}

4) Install the latest version of *Docker CE*:
{% highlight shell_session %}
# yum install docker-ce
{% endhighlight %}

5) Enable and start *Docker*:
{% highlight shell_session %}
# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.

# systemctl start docker
{% endhighlight %}

6) Verify that *docker* is installed correctly:
{% highlight shell_session %}
# docker run --rm hello-world
{% endhighlight %}

7) [Allow a non-privileged user to run Docker commands](https://docs.docker.com/engine/installation/linux/linux-postinstall/#manage-docker-as-a-non-root-user):
{% highlight shell_session %}
# groupadd docker
# usermod -aG docker dong
{% endhighlight %}

## nvidia-docker
[nvidia-docker](https://devblogs.nvidia.com/parallelforall/nvidia-docker-gpu-server-application-deployment-made-easy/) is essentially a wrapper around the *docker* command that transparently provisions a container with the necessary components to execute code on the GPU. It provides the two critical components needed for portable GPU-based containers:
* driver-agnostic CUDA images;
* a Docker command line wrapper that mounts the user mode components of the driver and the GPUs (character devices) into the container at launch.

1) [Install](https://github.com/NVIDIA/nvidia-docker/wiki) `nvidia-docker` & `nvidia-docker-plugin`:
{% highlight shell_session %}
# yum install https://github.com/NVIDIA/nvidia-docker/releases/download/v1.0.1/nvidia-docker-1.0.1-1.x86_64.rpm
{% endhighlight %}

2) Enable and start *nvidia-docker*:
{% highlight shell_session %}
# systemctl enable nvidia-docker
Created symlink from /etc/systemd/system/multi-user.target.wants/nvidia-docker.service to /usr/lib/systemd/system/nvidia-docker.service.
# systemctl start nvidia-docker
{% endhighlight %}

3) Verify that *nvidia-docker* is installed correctly:
{% highlight shell_session %}
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
{% endhighlight %}

### Upgrading to version 2.0
On December 1, 2017 I [upgraded nvidia-docker to version 2.0](https://github.com/NVIDIA/nvidia-docker/wiki/Installation-(version-2.0)).

1) Remove nvidia-docker 1.0:
{% highlight shell_session %}
# docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 docker ps -q -a -f volume={} | xargs -r docker rm -f
# yum remove nvidia-docker
{% endhighlight %}

2) [Install the repository](https://nvidia.github.io/nvidia-docker/):
{% highlight shell_session %}
# curl -s -L https://nvidia.github.io/nvidia-docker/centos7/x86_64/nvidia-docker.repo | tee /etc/yum.repos.d/nvidia-docker.repo
{% endhighlight %}

3) Install the *nvidia-docker2* package:
{% highlight shell_session %}
# yum install nvidia-docker2
{% endhighlight %}

4) Reload the *Docker* daemon configuration:
{% highlight shell_session %}
# pkill -SIGHUP dockerd
{% endhighlight %}

5) Verify that *nvidia-docker2* is installed correctly:
{% highlight shell_session %}
# nvidia-docker run --rm nvidia/cuda nvidia-smi
Sat Dec  2 05:17:41 2017
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.81                 Driver Version: 384.81                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 108...  Off  | 00000000:02:00.0 Off |                  N/A |
| 23%   32C    P0    58W / 250W |      0MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   1  GeForce GTX 108...  Off  | 00000000:03:00.0 Off |                  N/A |
| 23%   36C    P0    62W / 250W |      0MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   2  GeForce GTX 108...  Off  | 00000000:83:00.0 Off |                  N/A |
| 23%   28C    P0    58W / 250W |      0MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   3  GeForce GTX 108...  Off  | 00000000:84:00.0 Off |                  N/A |
| 23%   31C    P0    59W / 250W |      0MiB / 11172MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
{% endhighlight %}
