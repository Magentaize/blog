---
title: 记校园网 802.1X 连接频繁掉线
date: 2020-12-23 11:43:54
categories: Tech
---

<div class="banner-img">
    <img src="/images/2020/8021x-banner.png">
</div>

最近得知校园网部署了 802.1X 认证，以为终于摆脱了经常弹不出来网页还用了自签证书的Portal认证，把所有设备都配置到了新的 suep.1x 上。但是在近两天的使用过程里频繁遇到掉线问题，一通抓包最后发现是认证网关设置了2设备的最大连接限制，这就很不厚道了，如今谁还没有好几台设备呢。

<!--more-->

# 重复的SSID
因为我是用 AirportItlwm 来驱动黑苹果的 Intel 网卡，所以怀疑是驱动没有支持 EAP 认证，随后扒到了作者的回复[Question WPA2-Enterprise Support](https://github.com/OpenIntelWireless/itlwm/issues/101)，看起来似乎并没有实现。但是在 Windows 里也无法通过直接输入用户名密码的方式连接（iPhone 和 iPad 是可以直接连接的），这应该说明问题不在于 AirportItlwm。
在 macOS 和 Windows 上通过手动连接 SSID 并指定加密方式的方法成功添加了 802.1X 网络，不过在 macOS上有离奇的问题，出现了两个重复的 SSID，并且其中一个是无法连接的。
<img src="/images/2020/8021x-1.jpg">

# 随机掉线
## 系统日志
在网络掉线的时候，有两种表现。一种是菜单栏图标仍显显示连接，但是 QQ 已经提示 Connection Lost，另一种是菜单栏图标显示正在连接的动画，最后无法连接显示一个问号。
既然可以成功连接，那说明认证过程没有问题，只是由于不知名原因被迫断开。打开系统日志可以看到，eapolclient突然停止，并且在首次认证成功后会重复认证。
<img src="/images/2020/8021x-2.jpg">

## Wireshark 抓包
从抓包的结果可以看到 AP 控制器到网卡之间不存在丢包，EAP 与 TLS 过程也没有问题，但是在成功完成 EAP 认证之后，经过一段随机时间，AP 控制器会固定发送 Failure 的消息，随后 MAC 会自动重连。
<img src="/images/2020/8021x-3.jpg">

# 视奸网关
我以为是网卡的 MAC 地址不在认证网关的 whitelist 里，登录到网关里，发现 MacBook 的 MAC 地址确实存在，不过记录只有两条，缺少了 iPhone 的，并且在每次刷新的时候显示的两台设备还不一样。这就显然是，只保留了 LRU 的设备。
<img src="/images/2020/8021x-4.jpg">
最后在深澜软件的案例展示里，确实找到了设备限制的截图，那这就没得办法了，得接个 OpenWrt 的路由器了。
<img src="/images/2020/8021x-5.jpg">

PS: 网关里无感认证的增加MAC功能竟然是假的，手动添加之后虽然设备数量增加了，但是依然只有2个可以连接。