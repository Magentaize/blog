---
title: 重启 Ghost
date: 2018-03-21 06:43:09
categories: Note
---

之前的某一天，托管所有网站的Linux Server被爆破了，不过还好数据库和文件都没有被淦，之后重装系统的时候，就想着把CentOS换成Debian。换掉之后，其他的服务都和之前跑的一样好，但就是这个Ghost，遇到了各种莫名其妙的问题，什么给了权限还报错无权限，找不到文件，不识别挂载点balabala的。在Gayhub上问了开发团队，也不能给出有效的解决方案，百般无奈，在Debian上是跑不起Ghost了，只好试试在Win上。
<!--more-->

神奇的是，Win上能非常正常的运行并且没有任何毛病（但只能用ghost-cli@{version<=1.4.2}，因为更高版本的ghost-cli会调用一些chmod之类的命令）。那就很简单啦，只要在Nginx上转发到内网的Win Server就好了，在设置成监听0.0.0.0:2368的时候，ghost restart，奇迹发生了：

telnet里并不能看到LAN 2368端口打开了，我确定防火墙是没问题的，那么问题在哪呢？

嗯。。不用ghost-cli来启动了，just `node current/index.js`，接下来，更加神奇的事情发生了：

>INFO Ghost is running in development... 
INFO Listening on: 127.0.0.1:2368 
INFO Url configured as: http://0.0.0.0:2368/

???为什么还是监听了localhost?所以说，这个config是根本不生效的嘛，那host选项是做什么用的？这种事情只好黑一把js程序员了>.>

解决起来就好办了，在/core/server/config/index.js里nconf的constructor中覆盖掉默认设置就好了。

顺便这两天acme.sh支持了Let's Encrypt的野卡申请，一并申请并转移到了Cloudflare上。