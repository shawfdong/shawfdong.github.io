---
layout: post
title: Mounting a Subdirectory of CephFS on a CentOS 7 Client
tags: [Ceph, Linux]
---

In this post, we describe how to mount a subdirectory of CephFS on a machine running CentOS 7, particularly how to mount a subdirectory of our [Luminous Ceph filesystem]({{ site.baseurl }}{% post_url 2017-8-30-luminous-on-pulpos %}) on the 4-GPU workstation [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}).<!-- more --> For demonstration purpose, we'll restrict Hydra to mounting only the `hydra` directory of the CephFS, omitting the root directory. When you are given access to the CephFS, you'll have your own *Ceph Client* username, which may be different from your UNIX username. In order to mount your own directory of the CephFS on your own machine, you should replace all occurrence of `hydra` in the commands below with your own *Ceph Client* username.

* Table of Contents
{:toc}

## Add a Ceph Client User
So far, there is only one, default, Ceph Client user, *admin*, whose keyring is in the file `/etc/ceph/ceph.client.admin.keyring`.
{% highlight shell %}
[root@pulpo-admin ~]# ceph auth get client.admin
exported keyring for client.admin
[client.admin]
        key = AQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
{% endhighlight %}

On one of the monitor nodes, [add a new Ceph Client user](http://docs.ceph.com/docs/master/rados/operations/user-management/) *hydra*:
{% highlight shell %}
[root@pulpo-admin Pulpos]# ceph auth add client.hydra mon 'allow r' mgr 'allow r' mds 'allow rw' osd 'allow rw'
added key for client.hydra
{% endhighlight %}

Save user *hydra*'s key to a file `ceph.client.hydra.keyring`, in the *keyring* format:
{% highlight shell %}
[root@pulpo-admin Pulpos]# ceph auth get-or-create client.hydra -o ceph.client.hydra.keyring

[root@pulpo-admin Pulpos]# cat ceph.client.hydra.keyring
[client.hydra]
        key = AQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
{% endhighlight %}

Verify user *hydra*'s capabilities:
{% highlight shell %}
[root@pulpo-admin Pulpos]# ceph auth get client.hydra
exported keyring for client.hydra
[client.hydra]
        key = AQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
        caps mds = "allow rw"
        caps mgr = "allow r"
        caps mon = "allow r"
        caps osd = "allow rw"
{% endhighlight %}

Make a directory `hydra` on the CephFS:
{% highlight shell %}
[root@pulpo-dtn ~]# cd /mnt/pulpos/
[root@pulpo-dtn pulpos]# mkdir hydra
[root@pulpo-dtn pulpos]# chmod 1777 hydra
{% endhighlight %}

## Restrict CephFS Client Capabilities
Let's try to restrict Ceph Client *hydra*'s capabilities to only able to mount and work within the directory */hydra* of the Ceph filesystem, following [instructions in the official Ceph documentation](http://docs.ceph.com/docs/master/cephfs/client-auth/):
{% highlight shell %}
[root@pulpo-admin Pulpos]# ceph fs ls
name: pulpos, metadata pool: cephfs_metadata, data pools: [cephfs_data ]

[root@pulpo-admin ~]# ceph fs authorize pulpos client.hydra /hydra rw
Error EINVAL: key for client.hydra exists but cap mds does not match
{% endhighlight %}
Arghhh!! As it turns out, the capacities specified by `ceph auth` and those by `ceph fs authorize` must exactly match!  

Let's modify Ceph Client *hydra*'s capabilities and try again:
{% highlight shell %}
[root@pulpo-admin ~]# ceph auth caps client.hydra mon 'allow r' mgr 'allow r' osd 'allow rw pool=cephfs_data' mds 'allow rw path=/hydra'
updated caps for client.hydra

[root@pulpo-admin Pulpos]# ceph auth get client.hydra
exported keyring for client.hydra
[client.hydra]
        key = AQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
        caps mds = "allow rw path=/hydra"
        caps mgr = "allow r"
        caps mon = "allow r"
        caps osd = "allow rw pool=cephfs_data"

[root@pulpo-admin ~]# ceph fs authorize pulpos client.hydra /hydra rw
[client.hydra]
        key = AQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
{% endhighlight %}
and it works!

Similarly, we've added a Ceph Client *hb*, for mounting the CephFS on the hummingbird cluster:
{% highlight shell %}
[root@pulpo-dtn ~]# cd /mnt/pulpos/
[root@pulpo-dtn pulpos]# mkdir hb
[root@pulpo-dtn pulpos]# chmod 1777 hb

[root@pulpo-admin Pulpos]# ceph auth add client.hb mon 'allow r' mgr 'allow r' osd 'allow rw pool=cephfs_data' mds 'allow rw path=/hb'
[root@pulpo-admin Pulpos]# ceph fs authorize pulpos client.hb /hb rw

[root@pulpo-admin Pulpos]# ceph auth get-or-create client.hb -o ceph.client.hb.keyring
{% endhighlight %}

## Install ceph-fuse on CentOS 7 Client
We will not use the kernel CephFS driver, but *ceph-fuse*, to [mount CephFS]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}#mounting-cephfs-on-clients) on Hydra. Although the EPEL repo provide a *ceph-fuse* package, it is very outdated:
{% highlight shell %}
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
{% endhighlight %}
Ceph v0.80 is codenamed **Emperor**, which was [released in November 2013](http://docs.ceph.com/docs/master/releases/)! So we won't use *ceph-fuse* from the EPEL repo.

Install *yum-plugin-priorities*:
{% highlight shell %}
[root@hydra ~]# yum -y install yum-plugin-priorities
{% endhighlight %}

Add a [Yum repository for Luminous](http://docs.ceph.com/docs/master/install/get-packages/#rpm-packages) (`/etc/yum.repos.d/ceph.repo`):
{% highlight ini %}
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
{% endhighlight %}

Install *ceph-fuse*:
{% highlight shell %}
[root@hydra ~]# yum install ceph-fuse

[root@hydra ~]# ceph-fuse --version
ceph version 12.2.1 (3e7492b9ada8bdc9a5cd0feafd42fbca27f9c38e) luminous (stable)
{% endhighlight %}

## Mount a Subdirectory of CephFS
I) Create the directory `/etc/ceph` on Hydra:
{% highlight shell %}
[root@hydra ~]# mkdir /etc/ceph
{% endhighlight %}

II) Copy `ceph.conf` & `ceph.client.hydra.keyring` to */etc/ceph* on Hydra.

III) Change the permission of `/etc/ceph/ceph.client.hydra.keyring` so that only *root* can read and write it:
{% highlight shell %}
[root@hydra ~]# chmod 600 /etc/ceph/ceph.client.hydra.keyring
{% endhighlight %}

IV) Create the mountpoint, e.g., `/mnt/pulpos`:
{% highlight shell %}
[root@hydra ~]# mkdir /mny/pulpos
{% endhighlight %}

Now we are ready to mount the subdirectory `/hydra` of the CephFS, at the mountpoint `/mnt/pulpos` on Hydra. There are 3 options:

1) Manually run `ceph-fuse` command:
{% highlight shell %}
[root@hydra ~]# ceph-fuse -n client.hydra -m 128.114.86.4:6789 -r /hydra /mnt/pulpos
{% endhighlight %}
or more redundantly:
{% highlight shell %}
[root@hydra ~]# ceph-fuse -n client.hydra -m pulpo-mon01.ucsc.edu:6789,pulpo-mds01.ucsc.edu:6789,pulpo-admin.ucsc.edu:6789 -r /hydra /mnt/pulpos
{% endhighlight %}

2) To mount the subdirectory `/hydra` of the CephFS automatically on startup, we can add the following to `/etc/fstab` (see [http://docs.ceph.com/docs/luminous/cephfs/fstab/#fuse](http://docs.ceph.com/docs/luminous/cephfs/fstab/#fuse)):
{% highlight conf %}
none  /mnt/pulpos  fuse.ceph  ceph.id=hydra,ceph.client_mountpoint=/hydra,defaults,_netdev 0  0
{% endhighlight %}
the we can manually mount it with:
{% highlight shell %}
[root@hydra ~]# mount /mnt/pulpos/
{% endhighlight %}

3) Another option to automate mounting of the subdirectory `/hydra` of the CephFS is to use *systemd*. To take this route, we first need to modify the unit file `/usr/lib/systemd/system/ceph-fuse@.service`:
{% highlight conf %}
[Unit]
Description=Ceph FUSE client
After=network-online.target local-fs.target time-sync.target
Wants=network-online.target local-fs.target time-sync.target
Conflicts=umount.target
PartOf=ceph-fuse.target

[Service]
EnvironmentFile=-/etc/sysconfig/ceph
Environment=CLUSTER=ceph
ExecStart=/usr/bin/ceph-fuse -f --cluster ${CLUSTER} -n client.hydra -r /hydra %I
TasksMax=infinity
Restart=on-failure
StartLimitInterval=30min
StartLimitBurst=3

[Install]
WantedBy=ceph-fuse.target
{% endhighlight %}
Note that we've add the flags `-n client.hydra -r /hydra` to *ceph-fuse* in the *ExecStart* line of the unit file.

Reload systemd manager configuration (because we've made changes to the unit file for *ceph-fuse@.service*):
{% highlight shell %}
[root@hydra ~]# systemctl daemon-reload
{% endhighlight %}

Start the *ceph-fuse* service to mount the subdirectory `/hydra` of the CephFS, at the mountpoint `/mnt/pulpos`:
{% highlight shell %}
[root@hydra ~]# systemctl start ceph-fuse@/mnt/pulpos.service
{% endhighlight %}

To create a persistent mount point:
{% highlight shell %}
[root@hydra ~]# systemctl enable ceph-fuse.target

[root@hydra ~]# systemctl enable ceph-fuse@-mnt-pulpos
{% endhighlight %}
**NOTE** here the command must be `systemctl enable ceph-fuse@-mnt-pulpos`. If we run `systemctl enable ceph-fuse@/mnt/pulpos` instead, we'll get an error "Failed to execute operation: Unit name pulpos is not valid." However, when starting the service, we can run either `systemctl start ceph-fuse@/mnt/pulpos` or `systemctl start ceph-fuse@-mnt-pulpos`!

Lastly, we note the same bug in current version of Luminous (v12.2.1): when CephFS is mounted using *ceph-fuse*, the mount point doesn't show up in the output of `df`; and although we can list the mount point specifically with `df -h /mnt/pulpos`, the size of the filesystem is reported as 0!
{% highlight shell %}
[root@hydra ~]# df -h /mnt/pulpos
Filesystem      Size  Used Avail Use% Mounted on
ceph-fuse          0     0     0    - /mnt/pulpos
{% endhighlight %}
