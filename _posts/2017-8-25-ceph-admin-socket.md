---
layout: post
title: Ceph Admin Socket
tags: [Ceph, Storage, Provisioning]
---

The [Ceph admin socket](http://docs.ceph.com/docs/master/rados/operations/monitoring/#using-the-admin-socket) allows you to query a daemon via a socket interface.<!-- more -->

By default, Ceph sockets reside under `/var/run/ceph`. To access a daemon via the admin socket, you *must* login to the host running the daemon and use either of the following commands:
{% highlight shell_session %}
# ceph daemon {daemon-name} {command}
# ceph daemon {path-to-socket-file} {command}
# ceph --admin-daemon {path-to-socket-file} {command}
{% endhighlight %}

For example, the following are equivalent (on *pulpos-admin*, which runs a `ceph-mon` daemon):
{% highlight shell_session %}
[root@pulpo-admin ~]# ceph daemon mon.pulpo-admin help
[root@pulpo-admin ~]# ceph daemon /var/run/ceph/ceph-mon.pulpo-admin.asok help
[root@pulpo-admin ~]# ceph --admin-daemon /var/run/ceph/ceph-mon.pulpo-admin.asok help
{% endhighlight %}

You probably don’t want to restart Ceph daemon every time you make a change to your configuration. Fortunately, Ceph allows you to make changes to the configuration of a `ceph-osd`, `ceph-mon`, or `ceph-mds` daemon [at runtime](http://docs.ceph.com/docs/master/rados/configuration/ceph-conf/#ceph-runtime-config). The following reflects runtime configuration usage:
{% highlight shell_session %}
# ceph tell {daemon-type}.{id or *} injectargs --{name} {value} [--{name} {value}]
{% endhighlight %}
