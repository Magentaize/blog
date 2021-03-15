---
title: 在 .NET 3.5 中使用 WindowChrome 创建无边框窗口
date: 2018-04-25 10:04:44
categories:
 - Tech
tags:
 - WPF
---

关于使用无边框窗口并自绘控件来使GUI更符合个性化需求已经被讨论了太多，在.NET 4.5中借助WindowChrome来完成这一需求的范例也已经相当成熟，比如著名开源音乐播放器[Dopamine](https://github.com/digimezzo/Dopamine/)就是一例。但如果在某些特殊的要求下，只能使用.NET 3.5的情况下，很多基础部件都是缺失的，抛去`WindowStyle="None"`这种问题比较多的解决方法，最终比较好的方式是移植微软在.NET 4.5中为我们写好的库。
<!--more-->

那从哪里移植呢？显然WPF框架并不开源，但这不妨碍我们能读到WPF的源码。令人高兴的是，有人已经为我们反编译了Microsoft.Windows.Shell，在此处[WPF-Shell-Integration-Library](https://github.com/oysteinkrog/WPF-Shell-Integration-Library)。但是立刻就能发现当它被编译到.NET 3.5时，工作区的可视区域会发生计算错误，窗口的一部分会被裁掉（而编译到.NET 4.0时则不会发生），如果去读里面的WindowChrome.cs，关于布局的计算其实是非常麻烦的，并且涉及到了相当多的Win32部分。当然了，我们有理由相信巨硬在.NET 4.0修复了相关的错误，但是源代码受限制并不能编译到.NET 4.0上，如何解决呢？

显然去反编译PresentationFramework.dll@4.0.0.0是最迅速的，借助[dnSpy](https://github.com/0xd4d/dnSpy)，可以轻易地反编译出其中的WindowChrome，并替换掉WPF-Shell-Integration-Library中的WindowChrome.cs与WindowChromeWorker.cs这两个文件，之后再配合[WPFControls](https://github.com/digimezzo/WPFControls)就能实现在.NET 3.5中创建无边框窗口。
![dotnet35_with_windowchrome](/content/images/2018/04/dotnet35_with_windowchrome.png)

在反编译的WindowChromeWorker.cs中，可以看到_FixupFrameworkIssues、_FixupWindows7Issues、_FixupRestoreBounds等方法，说明在Windows7与WPF4及更高版本中，确实存在布局的问题，并且巨硬也确实修复了。
