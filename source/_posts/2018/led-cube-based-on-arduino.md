---
title: 用Arduino做的LED Cube
date: 2018-01-06 12:04:10
categories: Note
---

这学期选了一门叫做“基于开源硬件的电子制作”的选修课，一开始我还以为用的是Raspberry Pi或者Android Things之类的东西，没想到是单片机._. 其实之前也没怎么用过单片机，智能车无人机什么的对我来说又太难了，那就用Led Cube水一个交了作业么好了。
<!--more-->

最近也是在忙学校的FPGA实验展示和最后的期末考试，都没什么时间写点东西，心塞塞的。

直接上视频
<iframe width="560" height="315" src="https://www.youtube.com/embed/uZoHuQetNNY" frameborder="0" gesture="media" allow="encrypted-media" allowfullscreen></iframe>

一开始是计划控制到每个灯的，但是无耐PWM信号不能经过译码器扩展IO口来传递，想过外加电容之类的但是迫于水平不足，最后作罢，只实现了精确到行的细度控制，而且，这个mega2560的主频实在太低了，模拟PWM输出占用了大量的cpu时间结果都频闪了。之后查了下资料没用它原生的DigitalWrite函数，而是直接写到寄存器里，速度改善了很多，起码没有肉眼可见的延迟了。
