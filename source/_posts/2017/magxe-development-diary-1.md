---
title: Magxe 开发日志 1 - 选择轮子
date: 2017-09-24 13:16:51
categories:
 - Note
tags:
 - ASP.NET
---

## 什么是Magxe
很久之前，博客系统从WordPress换到了Ghost，本来是看到别人在用，第一感觉就是很简单，很存粹，Casper这个主题也比较有GNU style，但是真正用起来并不是很顺手，尤其是那个环境的部署，NodeJS的版本冲突，复杂的依赖，当Ghost升级到1.0版本的时候，更是甚至不提供平滑升级工具，Ghost-CLI这个管理工具对CentOS的支持差到令人发指，以及，语言的信仰。最后决定用C#重写一个轮子。
<!--more-->

## 设计数据库
可以说这是我的第一个独立项目，之前也没有过深入使用MySql的经验，性能优化什么的更是无从谈起，不过，先走出第一步再说。

根据巨硬的EF Core官方文档，若没有历史成本则推荐使用Code First，在设计之初就考虑将前台和后台做成两个模块，于是数据部分也应该单独抽出来作为Magxe.Data。

数据库结构大体与Ghost相同，但是去掉了部分无用字段，设置主键、数据类型与长度，创建DbContext等不再累述。若要使用EF框架，需在`OnConfiguring`方法中指定使用的数据库类型和数据库连接字符串，官方的MySql驱动并不能与EF Core 2.0兼容，在此换为巨硬官方的`Pomelo.EntityFrameworkCore.MySql`。然后在PMC中执行
```html
PM> Add-Migration Initialize -Project Magxe.Data
PM> Update-Database -Project Magxe.Data
```
即可敏捷完成数据库映射。

## 设计模板引擎
Ghost能做到什么？当然是科技以换壳为本。对于Magxe，我的设计是能够完美兼容已有的Ghost的主题的，一是再单独设计一套结构并没有相关的生态，二是Ghost所用的模板引擎express-hbs也已经足够成熟，功能也很完善。

于是找到了Handlebarsjs在.Net平台上的实现[Handlebars.Net](github.com/rexm/Handlebars.Net)，看ReadMe上基本的功能都有，包括`#if`这种逻辑控制。用起来也是比较简单，仅需两步，一编译，二赋值，即可敏捷渲染。

但是！它仅仅是一个渲染引擎，不包含任何的关于ASP.NET的相关接口，这意味着无法像Razor一样易用，也无法直接注册到DI容器中。因此，在正式开发Magxe之前，需要先实现一套能与ASP.NET Core配合使用的View Engine，于是，写了一个对Handlebars.Net的扩展：[Handlebars.Net.ViewEngine](https://github.com/Magentaize/Handlebars.Net.ViewEngine)。它的目标很简单，仅为了提供Handlebars.Net到ASP.NET MVC框架的一个中间桥梁，和熟悉的Razor一样易用而不需要每次都手动执行代码。
