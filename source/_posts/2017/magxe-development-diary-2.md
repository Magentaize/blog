---
title: Magxe 开发日志 2 - 完成首页
date: 2017-10-14 12:37:08
categories:
 - Note
tags:
 - ASP.NET
---

隔壁的czp说，我们Spring全家桶都是敏捷开发，敏捷渲染。但是，拒绝！我们C#要优雅开发，从容开发。本以为首页不涉及到和后台系统的交互是没什么复杂度的，然而并非如此，虽然首页很简单，但却要把大部分的基础组件都完成了才能展示出来，要对整个系统做一个初步的规划。
<!--more-->

![magxe-index](/content/images/2017/10/magxe-index.png)

## 设计模板引擎
当前版本的[Handlebars.Net.ViewEngine](https://github.com/Magentaize/Handlebars.Net.ViewEngine)和上一篇日志发布时变化了很多，主要是改变了初始化Options和注册Helper时的方式，并调整了一下结构，现在自定义的Helper也支持依赖注入了。
这个初始化方式的改变我觉得更应是思维方式的改变，即从IP(Imperative Programming/命令式编程)到FP(Functional Programming/函数式编程)的改变。过去的思想是把所有的变量都封装为对象，在需要时直接调用成员对象即可，而FP中所有的变量都是高阶函数，我不关心client如何设置一个对象，我只关心client能够**提供**一个**提供一个对象**的某种方法，我按照client说的去做就能得到恰当的对象，这赋予了程序更强大的动态能力。

具体到实现上来看，原来版本的`AddHandlebarsViewEngine`方法需要传递一个`HandlebarsViewEngineOptions`参数，而现在传递的是`Action<HandlebarsViewEngineOptions>`。
在注册Helper时，原来的方法是由client返回一个`IList<Helper>()`，这个列表中保存了所有Helper的实例，然后由ViewEngine在构造时添加到模板引擎中，需要写非常多的new()，既不方便，又不优雅。而现在传递一个`Action<IHelperList>`，现在由VE初始化一个列表，然后client返回一个告诉VE如何去修改这个列表的方法，VE去执行这个方法，执行后的列表就是client想要的样子。

比较有意思的是，之后我也看了一些支持DI的框架，其中的大多数也是用了这种方式来初始化。

## Helper中的懒加载
对于某些Helper，其数据并不从上下文而是从数据库中得到，并且这类Helper也并不是由ViewEngine直接调用，而是嵌套在View里。对于这类Helper，需要手动调用VE来渲染指定的模板，一开始我把`IHandlebarsViewEngine`当作一个Dependence写在了Helper的构造器参数里，结果dotnet和IIS Express进程直接crash，除了一个502.3错误什么都没有，最后调试才发现是循环依赖了`IHandlebarsViewEngine`导致的StackOverflow，那就只能在构造器里注入一个`IServiceProvider`，然后把`GetService<IHandlebarsViewEngine>`作为Factory去创建一个`Lazy<IHandlebarsViewEngine>`来处理掉循环依赖。

## 填[Handlebars.Net](github.com/rexm/Handlebars.Net)的坑
还是在上一篇中，提到了Handlebarsjs在.NET平台上的这个实现，看上去很美好，但实际上用起来并不是一个样子，我试着去寻找其他解析hbs模板的库，比如说[mustache-sharp](https://github.com/jehugaleahsa/mustache-sharp)，但是感觉用起来的问题更多，相比[express-hbs](https://github.com/barc/express-hbs)缺少了部分功能，最后还是选择debug Handlebars.Net。

主要的问题有两点
1.无法区分Variables和Block Helper
2.Variables和Helper执行顺序混乱

第一个问题是对于形如`{% raw %}{{#block_helper}}{{block_helper}}{{/block_helper}}{% endraw %}`这样的表达式，如果注册了Block Helper，会把中间的`{% raw %}{{block_helper}}{% endraw %}`认为也是Block Helper的起始Token，这个问题对词法分析器略加改动即可，多判断一个分支条件来生成正确的token，我也提交了一个原仓库的PR: [Fixes #226: Cannot resolve expression and block helper with the same name](https://github.com/rexm/Handlebars.Net/pull/227)。

第二个问题是对于形如`{% raw %}{{block_helper.value}}{{block_helper}}{% endraw %}`这样的表达式，如果注册了Helper，则无法正确对Inline Path赋值，这个问题发生在Token流的置换处理器中，稍后我应该也会去解决这个Bug。

## 截取摘要问题
首页不仅需要分页，还需要对每篇文章截取摘要，但是我找了一圈，除了用`XmlDocument`手动截取外没看到有什么公开的库可以用。而Ghost是用了[downsize](https://github.com/cgiffard/Downsize)这个库，多亏了C#对闭包和本地方法的强大支持，使得把一个JavaScript移植到.NET平台上并不困难，除了少数几个Substring和Regex需要略加修改，其余部分几乎可以直接按Js的版本来写。这个库开源在Github上[Downsize.Net](https://github.com/Magentaize/Downsize.Net)，并且可以使用NuGet敏捷安装。
```html
PM> Install-Package Downsize.Net
```

同时，HIGAN提到了一点，就是Ghost原生的截取方式并不具有弹性（[Ghost 调教日志 - 解决中文摘要的截取问题](http://blog.higan.me/chinese-excerpt-in-ghost/)），只是粗暴地选择了前33个单词，在Magxe中我选择了他的做法，选择第一个段落作为摘要。

## 自动ViewModel映射
在C#和Database之间，我们有Entity Framework来实现ORM，但是这个O如何去对应VM呢？也就是如何进行DAO到DTO的映射？[AutoMapper](https://github.com/AutoMapper/AutoMapper)可以帮我们做到这一点，只要我们在调用映射前配置好两种类型之间的映射关系，再调用
```csharp
TOut dest = IMapper.Map<TIn, TOut>(TIn object);
```
AutoMapper即可自动完成类型转换，并且支持递归转换，不用再每次手动实例化一个VM了，简洁而优雅。