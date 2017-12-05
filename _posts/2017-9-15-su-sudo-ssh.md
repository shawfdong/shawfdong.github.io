---
layout: post
title: Would su, sudo or ssh honor /sbin/nologin?
tags: [Linux, Security]
---

If a user's *login shell* is `/sbin/nologin`, would **su**, **sudo** or **ssh** honor it?<!-- more --> Let's find it out.

On a typical CentOS 7 installation, the *login shell* of user *adm* is `/sbin/nologin` (see `/etc/passwd`):
{% highlight conf %}
adm:x:3:4:adm:/var/adm:/sbin/nologin
{% endhighlight %}

## su
**su** apparently honors entries in `/etc/passwd`. If we try to use **su** to run a command with *adm*, it will fail, as expected.
{% highlight shell_session %}
[root@hydra ~]# su -c id adm
This account is currently not available.

[root@hydra ~]# su - -c id adm
This account is currently not available.
{% endhighlight %}

However, we can override the login shell in the password database, by supplying a shell (e.g., `-s /bin/bash`) in the CLI.
{% highlight shell_session %}
[root@hydra ~]# su -s /bin/bash -c id adm
uid=3(adm) gid=4(adm) groups=4(adm)

[root@hydra ~]# su -s /bin/bash -c pwd adm
/root

[root@hydra ~]# su - -s /bin/bash -c id adm
uid=3(adm) gid=4(adm) groups=4(adm)

[root@hydra ~]# su - -s /bin/bash -c pwd adm
/var/adm
{% endhighlight %}

If we give *adm* a password, we can even su to *adm* from an unprivileged user. For example:
{% highlight shell_session %}
[dong@hydra ~]$ su -s /bin/bash -c pwd adm
Password:
/home/dong

[dong@hydra ~]$ su - -s /bin/bash -c pwd adm
Password:
/var/adm
{% endhighlight %}

## sudo
By contrast, **sudo** doesn't honor `/sbin/nologin` in `/etc/passwd`:
{% highlight shell_session %}
[root@hydra ~]# sudo -u adm id
uid=3(adm) gid=4(adm) groups=4(adm)

[root@hydra ~]# sudo -s -u adm
bash-4.2$ pwd
/root
bash-4.2$ echo $HOME
/var/adm
bash-4.2$ echo $SHELL
/bin/bash
{% endhighlight %}
However, if we use the `-i` option to simulate initial login, sudo will run the shell specified by the password database entry of the target user as a login shell, in this case, `/sbin/nologin`:
{% highlight shell_session %}
[root@hydra ~]# sudo -u adm -i id
This account is currently not available.
{% endhighlight %}

## ssh
As expected, **ssh** does honor `/sbin/nologin` in the password database. If we change user *dong*'s login shell to `/sbin/nologin`, ssh will fail:
{% highlight shell_session %}
$ ssh -l dong hydra
Last login: Fri Sep 22 22:07:45 2017 from 128-211-52-208.lightspeed.mtryca.sbcglobal.net
This account is currently not available.
Connection to hydra.soe.ucsc.edu closed.

$ ssh -l dong hydra id
This account is currently not available.
{% endhighlight %}
