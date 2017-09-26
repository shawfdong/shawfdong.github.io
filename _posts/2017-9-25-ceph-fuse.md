---
layout: post
title: Mounting a Subdirectory of CephFS on a CentOS 7 Client
tags: [Ceph, Linux]
---

In this post, we describe how to mount a subdirectory of CephFS on a machine running CentOS 7, particularly how to mount a subdirectory of our [Luminous Ceph filesystem]({{ site.baseurl }}{% post_url 2017-8-30-luminous-on-pulpos %}) on the 4-GPU workstation [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}).<!-- more --> For demonstration purpose, we'll restrict Hydra to mounting only the *dong* directory of the CephFS, omitting the root directory. When you are given access to the CephFS, you'll have your own *Ceph Client* username, which may be different from your UNIX username. In order to mount your own directory of the CephFS on your own machine, you should replace all occurrence of *dong* in the commands below with your own *Ceph Client* username.

* Table of Contents
{:toc}

## Add a Ceph Client User
So far, there is only one Ceph Client user, *admin*, whose keyring is in the file `/etc/ceph/ceph.client.admin.keyring`.
```shell
[root@pulpo-admin ~]# ceph auth get client.admin
exported keyring for client.admin
[client.admin]
        key = AQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
```

On one of the monitor nodes, [add a new Ceph Client user](http://docs.ceph.com/docs/master/rados/operations/user-management/) *dong*:
```shell
[root@pulpo-admin ~]# ceph auth add client.dong mon 'allow r' mgr 'allow r' mds 'allow rw' osd 'allow rw'
added key for client.dong
```

Save user *dong*'s key to a file `ceph.client.dong.keyring`, in the *keyring* format:
```shell
[root@pulpo-admin ~]# ceph auth get-or-create client.dong -o ceph.client.dong.keyring

[root@pulpo-admin ~]# cat ceph.client.dong.keyring
[client.dong]
        key = AQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
```

Verify user *dong*'s capabilities:
```shell
[root@pulpo-admin ~]# ceph auth get client.dong
exported keyring for client.dong
[client.dong]
        key = AQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
        caps mds = "allow rw"
        caps mgr = "allow r"
        caps mon = "allow r"
        caps osd = "allow rw"
```

## Restrict CephFS Client Capabilities
Let's try to restrict Ceph Client *dong* to only mount and work within the directory */dong* of the Ceph filesystem, following [instructions in the official Ceph documentation](http://docs.ceph.com/docs/master/cephfs/client-auth/):
```shell
[root@pulpo-admin ~]# ceph fs ls
name: pulpos, metadata pool: cephfs_metadata, data pools: [cephfs_data ]

[root@pulpo-admin ~]# ceph fs authorize pulpos client.dong /dong rw
Error EINVAL: key for client.dong exists but cap mds does not match
```
Arghhh!! This should be filed as another bug in Ceph v12.2.0 (Luminous). I've tried various combinations of user caps and CephFS client caps; but they all resulted in the same error!  

## Install ceph-fuse on CentOS 7 Client
We will not use the kernel CephFS driver, but *ceph-fuse*, to [mount CephFS]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}#mounting-cephfs-on-clients) on Hydra. Although the EPEL repo provide a *ceph-fuse* package, it is very outdated:
```shell
[root@hydra ~]# yum info ceph-fuse
Available Packages
Name        : ceph-fuse
Arch        : x86_64
Epoch       : 1
Version     : 0.80.7
Release     : 0.8.el7
Size        : 1.4 M
Repo        : epel/x86_64
Summary     : Ceph fuse-based client
URL         : http://ceph.com/
License     : GPLv2
Description : FUSE based client for Ceph distributed network file system
```
Ceph v0.80 is codenamed **Emperor**, which was [released in November 2013](http://docs.ceph.com/docs/master/releases/)! So we won't use *ceph-fuse* from the EPEL repo.

Install *yum-plugin-priorities*:
```shell
[root@hydra ~]# yum -y install yum-plugin-priorities
```

Add a [Yum repository for Luminous](http://docs.ceph.com/docs/master/install/get-packages/#rpm-packages) (`/etc/yum.repos.d/ceph.repo`):
```ini
[Ceph]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/rpm-luminous/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=2

[Ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-luminous/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=2

[ceph-source]
name=Ceph source packages
baseurl=https://download.ceph.com/rpm-luminous/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=2
```

Install *ceph-fuse*:
```shell
[root@hydra ~]# yum install ceph-fuse

[root@hydra ~]# ceph-fuse --version
ceph version 12.2.0 (32ce2a3ae5239ee33d6150705cdb24d43bab910c) luminous (rc)
```

## Mount a Subdirectory of CephFS
I) Create the directory `/etc/ceph` on Hydra:
```shell
[root@hydra ~]# mkdir /etc/ceph
```

II) Copy `ceph.conf` & `ceph.client.dong.keyring` to */etc/ceph* on Hydra.

III) Change the permission of `/etc/ceph/ceph.client.dong.keyring` so that only *root* can read and write it:
```shell
[root@hydra ~]# chmod 600 /etc/ceph/ceph.client.dong.keyring
```

IV) Create the mountpoint, e.g., `/mnt/pulpos`:
```shell
[root@hydra ~]# mkdir /mny/pulpos
```

Now we are ready to mount the subdirectory `/dong` of the CephFS, at the mountpoint `/mnt/pulpos` on Hydra. There are 3 options:

1) Manually run `ceph-fuse` command:
```shell
[root@hydra ~]# ceph-fuse -n client.dong -m 128.114.86.4:6789 -r /dong /mnt/pulpos
```
or more redundantly:
```shell
[root@pulpo-dtn ~]# ceph-fuse -m pulpo-mon01:6789,pulpo-mds01:6789,pulpo-admin:6789 -r /dong /mnt/pulpos
```

2) To mount the subdirectory `/dong` of the CephFS automatically on startup, we can add the following to `/etc/fstab` (see [http://docs.ceph.com/docs/luminous/cephfs/fstab/#fuse](http://docs.ceph.com/docs/luminous/cephfs/fstab/#fuse)):
```conf
none  /mnt/pulpos  fuse.ceph  ceph.id=dong,ceph.client_mountpoint=/dong,defaults,_netdev 0  0
```
the we can manually mount it with:
```shell
[root@hydra ~]# mount /mnt/pulpos/
```

3) Another option to automate mounting of the subdirectory `/dong` of the CephFS is to use *systemd*. To take this route, we first need to modify the unit file `/usr/lib/systemd/system/ceph-fuse@.service`:
```ini
[Unit]
Description=Ceph FUSE client
After=network-online.target local-fs.target time-sync.target
Wants=network-online.target local-fs.target time-sync.target
Conflicts=umount.target
PartOf=ceph-fuse.target

[Service]
EnvironmentFile=-/etc/sysconfig/ceph
Environment=CLUSTER=ceph
ExecStart=/usr/bin/ceph-fuse -f --cluster ${CLUSTER} -n client.dong -r /dong %I
TasksMax=infinity
Restart=on-failure
StartLimitInterval=30min
StartLimitBurst=3

[Install]
WantedBy=ceph-fuse.target
```
Note that we've add the flags `-n client.dong -r /dong` to *ceph-fuse* in the *ExecStart* line of the unit file.

Reload systemd manager configuration (because we've made changes to the unit file for *ceph-fuse@.service*):
```shell
[root@hydra ~]# systemctl daemon-reload
```

Start the *ceph-fuse* service to mount the subdirectory `/dong` of the CephFS, at the mountpoint `/mnt/pulpos`:
```shell
[root@pulpo-dtn ~]# systemctl start ceph-fuse@/mnt/pulpos.service
```

To create a persistent mount point:
```shell
[root@pulpo-dtn ~]# systemctl enable ceph-fuse.target

[root@pulpo-dtn ~]# systemctl enable ceph-fuse@-mnt-pulpos
```
**NOTE** here the command must be `systemctl enable ceph-fuse@-mnt-pulpos`. If we run `systemctl enable ceph-fuse@/mnt/pulpos` instead, we'll get an error "Failed to execute operation: Unit name pulpos is not valid." However, when starting the service, we can run either `systemctl start ceph-fuse@/mnt/pulpos` or `systemctl start ceph-fuse@-mnt-pulpos`!

Lastly, we note the same bug in current version of Luminous (v12.2.0): when CephFS is mounted using *ceph-fuse*, the mount point doesn't show up in the output of `df`; and although we can list the mount point specifically with `df -h /mnt/pulpos`, the size of the filesystem is reported as 0!
```
[root@pulpo-dtn ~]# df -h /mnt/pulpos
Filesystem      Size  Used Avail Use% Mounted on
ceph-fuse          0     0     0    - /mnt/pulpos
```
