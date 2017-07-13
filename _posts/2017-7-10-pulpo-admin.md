---
layout: post
title: Pulpo-Admin
tags: [Ceph, Storage, Linux]
---

The admin node of our Ceph storage cluster **Pulpos** is **pulpo-admin**.<!-- more -->

## CentOS 7 minimal install

### Update CentOS 7
{% highlight shell %}
[root@pulpo-admin ~]# yum update
{% endhighlight %}
Then reboot.

### Selinux
Disable Selinux by changing the following line in **/etc/selinux/config**:
{% highlight conf %}
SELINUX=enforcing
{% endhighlight %}
to:
{% highlight conf %}
SELINUX=disabled
{% endhighlight %}
Then reboot the node.

### sshd
Next we modify **/etc/ssh/sshd_config** to configure the OpenSSH server.
* Disable password authentication by changing the following line:
  ```conf
  PasswordAuthentication yes
  ```
  to
  ```conf
  PasswordAuthentication no
  ```
* Disable GSSAPI authentication by changing the following line:
  ```conf
  GSSAPIAuthentication yes
  ```
  to:
  ```conf
  GSSAPIAuthentication no
  ```
* Then restart sshd
  ```shell
  [root@pulpo-admin ~]# systemctl restart sshd
  ```
