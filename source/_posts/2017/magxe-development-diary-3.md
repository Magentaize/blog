---
title: Magxe 开发日志 3 - 实现文章页
date: 2017-10-15 06:41:11
categories:
 - Note
tags:
 - ASP.NET
---

在上一篇开发日志写完后不久，做了几件微小的工作就完成了文章页的模板渲染，于是版本号终于能进入0.0.1 Alpha了，实际上关于Handlebars Helper的改变很简单，都是基础的CURD，不过由于Handlebars.Net本身的一个bug导致了进度的缓慢。这里还是想吐槽一下.NET的生态问题，.NET技术栈好吗？当然好啦，Entity Framework、ASP.NET可以对扛Java三大件，PostSharp、Newtonsoft.Json也可以渗入程序的各个方面并且API友好，但是除了这些大型应用呢？遗憾的是个人需求无法找到能够满足需求的依赖，可能有多个不同方面的实现，但是无法汇聚为一个开箱即用的框架。
<!--more-->

## contentFor 块级 Helper
这个功能并不包含在原生Handlebars.js中，而只是Node后端开发为了增强模板复用而新增的Helper，作为一个Express框架的扩展：[express-handlebars](https://github.com/ericf/express-handlebars)。实现起来也非常简单，也就是新增两个关键字“contengFor”和“block”，一个负责将元素压入堆栈，而另一个负责将堆栈中的模板弹出。
如果直接在源码中集成这个功能，是比较简单的，甚至还可以进一步优化由模板生成的AST，但似乎Handlebars.Net的作者并不希望为其增加更多的功能，见[Any plan about adding support of 'block' and 'contentFor'?](https://github.com/rexm/Handlebars.Net/issues/225)。

不过用注册Helper的方式来实现也并不复杂，只需要建立一个公共的Dictionary就可以啦。
```csharp
internal static class BlockAndContentForHelper
{
    internal static readonly Dictionary<string, string> Blocks = new Dictionary<string, string>();
}

internal class ContentForHelper: HandlebarsBaseHelper {
    public ContentForHelper() : base("contentFor", HelperType.HandlebarsBlockHelper) {}

    public override void HandlebarsBlockHelper(TextWriter output, HelperOptions options, dynamic context, params object[] arguments) {
        var key = arguments[0].CastTo < string > ();
        var sb = new StringBuilder();
        using(var sw = new StringWriter(sb)) {
            options.Template(sw, null);
        }
        Blocks[key] = sb.ToString();
	}
}

internal class BlockHelper : HandlebarsBaseHelper
{
    public BlockHelper() : base("block", HelperType.HandlebarsHelper)
    {}

    public override void HandlebarsHelper(TextWriter output, dynamic context, params object[] arguments)
    {
        var key = arguments[0].CastTo<string>();
        if (Blocks.ContainsKey(key))
        {
            output.WriteSafeString($"{Blocks[key]}\n");
            Blocks[key] = string.Empty;
        }
        else
        {
            Blocks[key] = string.Empty;
        }
    }
}
```
这里我敏捷地用了一个静态属性来作为容器是非常有bad smell的，目前我无法得知在多并发环境下会导致什么样的结果，如果未来这里出现问题将考虑其他实现方式。

## Variables 和 Helper 执行顺序混乱
这是Handlebars.Net中的一个bug，在[Added ViewEngine / Layout support](https://github.com/rexm/Handlebars.Net/pull/76)有人为其增加了渲染Layout的功能，但是由于模板嵌套的情况过于复杂，并没有直接使新增的VM继承原有的，而是直接增加了一个DynamicViewModel类，但是在后期绑定中又没有对模板的内容做出判断，导致无法反射出VM中的却存在的内容，暂时的解决方案是每次绑定都要判断VM的类型，虽然不够优雅，但是要解决这个问题可能需要大面积地重构，也就先放一放。详见[Fixes error binding context in layout](https://github.com/rexm/Handlebars.Net/pull/229/files)。

## 让 Helper 基于接口调用
暂时还没有看完express-handlebars的源码，尚不清楚在JS里是如何解决复杂的模板嵌套问题的，而Ghost主题会在不同的模板中多次调用`@[PropertyName]`来请求一个数据，这就令Helper无法对当前渲染的上下文作出假设，而静态类型语言在运行时调用同名属性又非常不优雅，因此让所有VM去实现和Helper以及View相协定的接口，以避免脏代码的出现。
