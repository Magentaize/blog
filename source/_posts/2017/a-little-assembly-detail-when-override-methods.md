---
title: 一个方法重写(override)的程序集小细节
date: 2017-09-23 02:32:29
categories:
 - Tech
tags:
- C#
---

在写一个Helper的基类时，很简单的定义了一个如下的抽象类：
<!--more-->
```csharp
public abstract class BaseHelper
{
    public string HelperName { get; }

    public virtual void Helper(TextWriter output, dynamic context, params object[] arguments)
    {
        throw new NotImplementedException();
    }

    public virtual void BlockHelper(TextWriter output, HelperOptions options, dynamic context, params object[] arguments)
    {
        throw new NotImplementedException();
    }
}
```
而具体实现也非常简单，无非就是重写一下而已：
```csharp
internal class TagsHelper : BaseHelper
{
    public override void BlockHelper(TextWriter output, HelperOptions options, dynamic context, params object[] arguments){}

    public override void Helper(TextWriter output, dynamic context, params object[] arguments){}
}
```
但是并不能编译通过，一直提示`BlockHelper`这个方法no suitable method found to override。经过反复对比，并不能找到这个重写方法和基方法有什么区别，并且这也是Resharper自动生成的，所以函数签名应该是没有问题的。

不过，另一个方法`Helper`却没有任何问题，那么问题估计发生在`HelperOptions`这个类型上面。这个类是另外一个外部依赖中的，而抽象类引用的是nuget包，而子类引用的是用源码编译的版本，导致了在引用时发生程序集冲突。

而如何能知道究竟是程序集问题还是重写的方法不对呢？当没有可重写的虚方法时，`override`和方法名会一同被标记红波浪线，而程序集错误只会标记方法名。

不过，如果是程序集问题导致的无法重写，为什么不提示是程序集问题呢？却只提示无可重写的方法真的是有点难以捉摸。