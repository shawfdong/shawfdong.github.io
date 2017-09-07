---
layout: post
title: Tentakel
tags: [Linux, Provisioning]
---

**Tentakel** is a program for executing the same command on many hosts in parallel using various remote methods.<!-- more --> Tentakel is bundled with the [Rocks](http://www.rocksclusters.org/wordpress/) cluster toolkit; so it should be familiar to Rocks users. In my humble option, Tentakel has been largely superseded by [Ansible]({{ site.baseurl }}{% post_url 2017-7-13-ansible %}), which can easily do everything Tentakel can, and many more things that Tentakel can't.

## Installing Tentakel
[https://github.com/sfermigier/tentakel] is written in Python. So the easiest way to install Tentakel is to use [pip](https://pip.pypa.io/en/stable/).

1. Install [pip](https://pip.pypa.io/en/stable/installing/):
```shell
# python get-pip.py
```
2. Install Tentakel:
```shell
# pip install tentakel
```

## Configuring Tentakel
Create a Tentakel configuration file `/etc/tentakel.conf`:
```conf
group default ()
  @ceph

group ceph ()
  @client
  @mon
  @mds
  @osd

group client ()
  +192.168.1.2

group mon ()
  +192.168.1.3

group mds ()
  +192.168.1.4

group osd ()
  +192.168.1.5
  +192.168.1.6
  +192.168.1.7
```

## Testing Tentakel
Let's do some simple tests:
```shell
# tentakel hostname
### 192.168.1.2(stat: 0, dur(s): 0.13):
pulpo-dtn.ucsc.edu
### 192.168.1.4(stat: 0, dur(s): 0.13):
pulpo-mds01.ucsc.edu
### 192.168.1.5(stat: 0, dur(s): 0.14):
pulpo-osd01.ucsc.edu
### 192.168.1.5(stat: 0, dur(s): 0.13):
pulpo-osd01.ucsc.edu
### 192.168.1.5(stat: 0, dur(s): 0.14):
pulpo-osd01.ucsc.edu
### 192.168.1.3(stat: 0, dur(s): 0.13):
pulpo-mon01.ucsc.edu

# tentakel -g osd uptime
### 192.168.1.5(stat: 0, dur(s): 0.14):
 14:24:54 up 3 days, 22 min,  0 users,  load average: 0.09, 0.22, 0.19
### 192.168.1.5(stat: 0, dur(s): 0.13):
 14:24:54 up 3 days, 22 min,  0 users,  load average: 0.09, 0.22, 0.19
### 192.168.1.5(stat: 0, dur(s): 0.14):
 14:24:54 up 3 days, 22 min,  0 users,  load average: 0.09, 0.22, 0.19
 ```
