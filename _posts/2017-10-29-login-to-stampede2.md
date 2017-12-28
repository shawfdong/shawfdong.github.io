---
layout: post
title: Login to Stampede2
tags: [HPC]
---

There are 2 ways to login to TACC supercomputer [Stampede2](https://www.tacc.utexas.edu/systems/stampede2) supercomputer.<!-- more -->

## via XSEDE SSO Hub
0) [Set up Duo for XSEDE user portal SSO account](https://portal.xsede.org/mfa)

1) Login to [XSEDE SSO (Single Sign On) Hub](https://portal.xsede.org/single-sign-on-hub):
{% highlight shell_session %}
$ ssh -l shawdong login.xsede.org
{% endhighlight %}

2) Login to Stampede2:
{% highlight shell_session %}
[shawdong@ssohub ~]$ gsissh stampede2
{% endhighlight %}

## direct SSH
0) [Set up Multifactor Authentication at TACC](https://portal.tacc.utexas.edu/tutorials/multifactor-authentication)

1) Login to Stampede2:
{% highlight shell_session %}
$ ssh shawdong@stampede2.tacc.utexas.edu
{% endhighlight %}
