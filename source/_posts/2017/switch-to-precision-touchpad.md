---
title: Switch to Precision Touchpad
date: 2017-10-22 02:54:01
categories: Note
---

I have to say the offical touchpad driver of ELAN is much more better than others, however, after I've tried Ultrabook's glass touchpad of Lenovo, its smooth gesture gives me a deep impression.

Honestly, this ELAN driver also provides many gestures, some of them are more powerful than Windows built-in such as taping twice to hold the right button and once more to release but it cannot give me a silky experience because it only simulates a PS/2 mouse.
<!--more-->

At the meantime of the release of Windows 10, it intruduces a new standard "Precision Touchpad" which can provide a completely different feel. BUT, ELAN and MSI performance so indifferent to this even they cannot provide a new version driver!

There is always a way out, in this video [Does Your Laptop Trackpad Suck? Upgrade it for Free!](https://www.youtube.com/watch?v=f2rfwR-IV-c) Dave Lee introduced that we can update our toupad driver manually so I'll take a try.

Not as same as him, I download the driver at [Microsoft Update Catalog](https://www.catalog.update.microsoft.com/) and search "elan wdf". Open device manager, update elan driver manually and reboot. After reboot, it will detect a new **Human Interface Device** and redo the previous operation. Now, I have a new Precision Touchpad.
