---
title: 绕过 Teamspeak 5 账号徽章验证
date: 2020-06-28 09:18:20
categories:
 - Tech
---
<div class="banner-img">
    <img src="/images/2020/teamspeak-banner.jpg">
</div>

Teamspeak 5 已经内测了近两年了，没有公开地进行测试，而且 Teamspeak 3 的几次版本更新似乎破坏了不同客户端的响度控制。在同一个服务器中，不同版本与 OS 组合会产生明显的音量过大或过小的问题，极其影响使用，而TS5中内置了响度平衡功能，官方却用Badge Code将普通用户拒之门外。由于TS5使用了CEF技术来构建客户端，这给了绕过验证的可乘之机。
<!--more-->

首先需要一个老版本的TS5安装包，比如这里的 [https://ciphers.pw/threads/teamspeak-5-fully-working.8348](https://ciphers.pw/threads/teamspeak-5-fully-working.8348)，因为更高版本的客户端将部分逻辑放在了服务端，如果登录的账号没有内测资格，那么会无法使用网络功能。

然后根据zip档内的readme，同意EULA，替换db文件，这时候打开TS5会由于某种时间或网络验证要求重新登录账号。但是根据测试，此时客户端功能已经work，而登录账号的UI只是一个顶层div，因此只需让其不再显示这个div即可。在html\client_ui\main_xxxx.js中搜索函数`pushFullscreenQueueItem`，这出现在一个短路判断后：
```javascript
this.pushFullscreenQueueItem({identifier:"new-password-needed",component:q})
```
由于短路的前部分不具有side effects，因此删除该函数的调用即可。

PS. 在同一个服务器内，如果有多名用户使用了相同的db文件来连接，会遇到Unique ID冲突的错误，在TS5的Identities设置中新建一个Identity即可。

PS2. 为了防止TS5客户端的自动升级，需要将127.0.0.1 update.teamspeak.com添加到hosts。