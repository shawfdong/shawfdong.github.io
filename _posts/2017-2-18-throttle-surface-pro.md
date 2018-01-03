---
layout: post
title: Throttling Surface Pro 3
tags: [Windows 10]
---

In factory condition, the fans of my Surface Pro 3 ran almost non-stop, emitting a very audible crackling sound, even when the system was under very light load!<!-- more -->  In this post, we document how I throttled the CPU speed, and essentially turned my Surface Pro 3 into a passively cooled tablet.

Microsoft introduced a new feature called **Connected Standby** in Windows 8, which has been expanded to **Modern Standby** in Windows 10. There are some claimed [advantages of using Modern Standby](https://surfacetip.com/how-to-unlock-power-plans-on-surface-devices/):
* Instant On/Off
* Background activity while the system is “off”
* Simplified wake story

On every Surface device, **Connected Standby** / **Modern Standby** is enabled and a predefined power plan called **Balanced** is loaded by default. However, if we clicked "Change plan settings" then "Change advanced power settings", we found that very few advanced settings were available!

In order to unlock the hidden advanced settings, we need to temporarily disable **Connected Standby** / **Modern Standby**. Open **Register Editor** (regedit), change the value for the key:
{% highlight conf %}
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Power\CsEnabled
{% endhighlight %}
from **1** to **0**.

Reboot!

In Control Panel > Hardware and Sound > Power Options, click "Create a power plan" in the left panel, and create a new power plan named **Tablet Mode**.

Select the **Tablet Mode** power plan, click "Change plan settings" then "Change advanced power settings". All hidden settings are now unlocked! Make the following changes:
* Change the value for "Processor power management" > "System cooling policy" to **Passive**, for both "On Battery" and "Plugged in";
* Change the value for "Processor power management" > "Maximum Processor state" to **60%** for "On Battery", and to **50%** for "Plugged in".

Now we can reenable **Connected Standby** / **Modern Standby**. Open **Register Editor** (regedit), change the value for the key:
{% highlight conf %}
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Power\CsEnabled
{% endhighlight %}
from **0** to **1**.

Reboot!

We can verify that the CPU speed is indeed throttled even when **Connected Standby** / **Modern Standby** is turned on, by running a system profiling utility such as [CPU-Z](https://www.cpuid.com/softwares/cpu-z.html).
