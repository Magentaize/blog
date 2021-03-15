---
title: '"Hey Cortana" 会拖慢整个Windows'
date: 2017-08-17 15:38:44
categories:
 - Note
---

从某天起，整个系统的响应都变得极为迟缓，具体表现为explorer打开一个文件夹要经过3秒才能显示，打开taskmgr甚至要等待10秒，Visual Studio加载pdb非常卡。
<!--more-->

当全新安装之初，是没有这些问题的，我排查了debugger、QPCore、AHK等都不能发现问题，但是在不知某处的网站里，发现关闭"Hey Cortana"可以解决这个问题，的确如此。

早在2016年微软社区里就有人反映了这个问题['Hey Cortana' slow down Windows](https://answers.microsoft.com/en-us/surface/forum/surfpro4-surfperf/hey-cortana-slow-down-windows/9b5c105d-50c2-4bdd-a5bc-231d626b05bb?auth=1)，显然，这又是一个微软至今没有修复的bug。
