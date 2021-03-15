---
title: 吐槽 WPF 的窗口闪烁问题
date: 2017-07-30 03:54:46
categories:
 - Tech
tags:
- WPF
---

在一个项目里用到了类似“吸附式窗口”的东西，就是子窗体附着在父窗体周围并跟随父窗体移动。有个用户反应在 Show 了子窗体之后，移动父窗体会有明显的延迟和闪烁感觉。

该问题发生时 UI 线程的 `GetMessageW` 方法 CPU 占用率极高，起初估计是 UI 线程无法及时处理消息，但是后来发现如果不设置窗体的 `Top` 属性则不会发生闪烁，于是怀疑是 WPF 的依赖属性造成的性能下降，考虑直接使用 WinAPI 来改变窗体位置。之后尝试了几种方法，`MoveWindow`、`SetWindowPos`、父窗体捕获 `WM_SIZE` 消息、父窗体向子窗体发送 `WM_SIZE` 消息，卡顿确实有很大程度上的缓解，但是闪烁问题依然存在。
<!--more-->

最后发现，这是 WPF 本身的问题，当改变 `Top` 和 `Left` 属性时，会导致面板内元素重新布局，又因为父窗体是一个无边框窗体，DWM 无法绘制窗体的边框以容纳控件，就导致了窗口边缘的闪烁问题。然而知道了原因也没有什么用，因为微软给出的解决方案是 *Closed as Won't Fix*，时隔八年，这个问题依然没有被修复。

<font color='grey'>引用：
[Custom Resizing of System.Windows.Window flickers](https://connect.microsoft.com/VisualStudio/feedback/details/522441/custom-resizing-of-system-windows-window-flickers)
[WPF screen flickering while resizing](https://connect.microsoft.com/VisualStudio/feedback/details/652771/wpf-screen-flickering-while-resizing)
</font>