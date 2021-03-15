---
title: "[译]在 ASP.NET Core 中创建自定义模板引擎"
date: 2017-09-19 04:45:41
categories:
 - Tech
tags:
- ASP.NET
description: 几个月前，Taylor Mullen在"The Monsters Weekly"中谈到了关于ASP.NET Core里的Razor引擎。在采访中的某个时刻有人指出，在MVC的设计中，我们可以轻易地向框架内插入一个新的模板引擎。还有人指出，实现一个模板引擎实际上是一件非常复杂的事情。这令我们去思考，如果我们有一个现有的模板引擎的话应该怎么办？向MVC框架中插入一个新的模板引擎到底有多简单？
---

原文来自[Creating a New View Engine in ASP.NET Core](https://www.davepaquette.com/archive/2016/11/22/creating-a-new-view-engine-in-asp-net-core.aspx)
>原作者保留一切权利

### 疯狂的想法
几个月前，Taylor Mullen在"The Monsters Weekly"中谈到了关于ASP.NET Core里的Razor引擎。在采访中的某个时刻有人指出，在MVC的设计中，我们可以轻易地向框架内插入一个新的模板引擎。还有人指出，实现一个模板引擎实际上是一件非常复杂的事情。这令我们去思考，如果我们有一个现有的模板引擎的话应该怎么办？向MVC框架中插入一个新的模板引擎到底有多简单？

### 寻找一个代替品
我们想选择和Razor不太一样的东西，就像是Simon推荐的在Express框架中非常流行的[Pug](https://pugjs.org/api/getting-started.html)。而在语法方面，Pug和Razor之间略有不同，Pug使用空格来表示嵌套的元素并略去了尖括号，就像下面这样：
```html
div
    a(href='google.com') Google
```
这一段将会生成这种HTML：
```html
<div>
    <a href="google.com">Google</a>
</div>
```

### 在ASP.NET Core中使用Pug
我们所面临的第一个问题是如何在ASP.NET Core程序中编译Pug模板，而Pug是一个基于JavaScript的模板引擎，但是我们只有一天的时间，所以把Pug移植到C#是不太可能的。

我们的第一个想法是使用Edgejs来调用Pug的Compile方法，一些快速原型向我们展示出这是可以做到的，但是Edgejs并不支持.NET Core，这将引导我们去使用由ASP.NET Core团队编写的包“JavaScriptServices”，特别是那个可以让我们在ASP.NET Core程序中轻易调用JavaScript模块的“Node Services”包。

令我们惊喜的是，这个包不仅仅能够工作，而且非常易用！先创建一个叫做pugcompile.js的文件。
```javascript
var pug = require('pug');

module.exports = function (callback, viewPath, model) {
	var pugCompiledFunction = pug.compileFile(viewPath);
	callback(null, pugCompiledFunction(model));	
};
```

多亏了Node Services，在C#中调用JavaScript是如此简单。假设`model`是我们想绑定到模板的ViewModel，`mytemplate.pug`是包含Pug模板的文件。
```csharp
var html = await _nodeServices.InvokeAsync<string>("pugcompile", "mytemplate.pug", model);
```
现在我们已经证明这么做是可行的，是时候创建一个模板引擎并将其与MVC框架整合的时候的了。

### 创建Pugzor模板引擎
我们出于好玩决定把我们的模板引擎命名为Pug和Razor的组合：Pugzor。当然了这并不是很有意义，因为它和Razor并没有关系。

始终要记得，我们的目标是在一天之内实现一个模板引擎，我们希望敏捷开发。在花了点时间看了一下MVC的源代码之后，我们确定下来需要实现`IViewEngine`接口和自定义的`IView`。

`IViewEngine`负责根据`ActionContext`和`ViewName`来确定`View`的位置。当`Controller`返回一个`View`时，它实际上是`IViewEngine`中一个用于根据约定来返回`View`的`FindView`方法。`FindView`方法返回一个`ViewEngineResult`，`ViewEngineResult`是一个拥有两个属性的简单的类，其中一个是`bool Success`属性，用于表明是否找到了一个View，另一个是包含该View（如果找到）的`IView View`属性。
```csharp
/// <summary>
/// 定义模板引擎契约。
/// </summary>
public interface IViewEngine
{
    /// <summary>
    /// 使用<paramref name="context"/>中视图的位置信息来查找指定<paramref name="viewName"/>的视图。
    /// </summary>
    /// <param name="context">The <see cref="ActionContext"/>.</param>
    /// <param name="viewName">视图的名字。</param>
    /// <param name="isMainPage">确定找到的页面是否是操作的主页面。</param>
    /// <returns>找到的视图的<see cref="ViewEngineResult"/>。</returns>
    ViewEngineResult FindView(ActionContext context, string viewName, bool isMainPage);

    /// <summary>
    /// Gets the view with the given <paramref name="viewPath"/>, relative to <paramref name="executingFilePath"/>
    /// unless <paramref name="viewPath"/> is already absolute.
    /// </summary>
    /// <param name="executingFilePath">The absolute path to the currently-executing view, if any.</param>
    /// <param name="viewPath">The path to the view.</param>
    /// <param name="isMainPage">Determines if the page being found is the main page for an action.</param>
    /// <returns>The <see cref="ViewEngineResult"/> of locating the view.</returns>
    ViewEngineResult GetView(string executingFilePath, string viewPath, bool isMainPage);
}
```
我们决定使用与Razor相同的View位置约定，也就是说，View位于`Views/{ControllerName}/{ActionName}.pug.`中。
以下是PugzorViewEngine的`FindView`方法的简化版本：
```csharp
public ViewEngineResult FindView(
    ActionContext actionContext,
    string viewName,
    bool isMainPage)
{
    var controllerName = GetNormalizedRouteValue(actionContext, ControllerKey);
 
    var checkedLocations = new List<string>();
    foreach (var location in _options.ViewLocationFormats)
    {
        var view = string.Format(location, viewName, controllerName);
        if(File.Exists(view))
            return ViewEngineResult.Found("Default", new PugzorView(view, _nodeServices));
        checkedLocations.Add(view);
    }
    return ViewEngineResult.NotFound(viewName, checkedLocations);
}
```
你可以在[Github](https://github.com/AspNetMonsters/pugzor/blob/master/Pugzor.Core/PugzorViewEngine.cs)上找到完整的实现。

接下来，创建一个`PugzorView`类来实现`IView`接口，`PugzorView`接受一个pub模板的路径和一个`INodeServices`实例。当MVC框架需要渲染视图时，它会调用`IView`的`RenderAsync`方法，在这个方法中，调用`pugcompile`并将生成的HTML写入视图上下文。
```csharp
public class PugzorView : IView
{
    private string _path;
    private INodeServices _nodeServices;

    public PugzorView(string path, INodeServices nodeServices)
    {
        _path = path;
        _nodeServices = nodeServices;
    }

    public string Path
    {
        get
        {
            return _path;
        }
    }

    public async Task RenderAsync(ViewContext context)
    {
        var result = await _nodeServices.InvokeAsync<string>("./pugcompile", Path, context.ViewData.Model);
        context.Writer.Write(result);
    }
}
```
唯一剩下的就是配置MVC来使用我们的新模板引擎。一开始，我们认为在将MVC添加到服务集合中时可以很方便地使用`AddViewOptions`扩展方法来添加一个新的模板引擎。
```csharp
// Startup.cs
services.AddMvc()
        .AddViewOptions(options =>
            {
                options.ViewEngines.Add(new PugzorViewEngine(nodeServices));
            });
```
但是这也是令我们困惑的地方，因为我们无法在`Startup.ConfigureServices`方法中为`ViewEngines`集合添加`PugzorViewEngine`的具体实例，因为模板引擎的构造函数需要依赖注入。`PugzorViewEngine`依赖了`INodeServices`并且我们希望这个参数能够由ASP.NET的DI框架注入。不过幸运的是，对Razor无所不知的大师Taylor Mullen会向我们展示注册模板引擎的正确方法。

将模板引擎添加到MVC中的推荐方法是创建一个实现了`IConfigureOptions <MvcViewOptions>`接口的“安装”类，这个类通过构造器注入得到了我们的`IPugzorViewEngine`实例对象，而在`ConfigureServices`方法中，这个实例对象会被添加到`MvcViewOptions`的模板引擎列表中。
```csharp
public class PugzorMvcViewOptionsSetup : IConfigureOptions<MvcViewOptions>
{
    private readonly IPugzorViewEngine _pugzorViewEngine;

    /// <summary>
    /// 初始化一个<see cref="PugzorMvcViewOptionsSetup"/>的新实例。
    /// </summary>
    /// <param name="pugzorViewEngine">The <see cref="IPugzorViewEngine"/>.</param>
    public PugzorMvcViewOptionsSetup(IPugzorViewEngine pugzorViewEngine)
    {
        if (pugzorViewEngine == null)
        {
            throw new ArgumentNullException(nameof(pugzorViewEngine));
        }

        _pugzorViewEngine = pugzorViewEngine;
    }

    /// <summary>
    /// 配置<paramref name="options"/>来调用<see cref="PugzorViewEngine"/>。
    /// </summary>
    /// <param name="options">The <see cref="MvcViewOptions"/> to configure.</param>
    public void Configure(MvcViewOptions options)
    {
        if (options == null)
        {
            throw new ArgumentNullException(nameof(options));
        }

        options.ViewEngines.Add(_pugzorViewEngine);
    }
}
```
现在我们需要做的是在`Startup.ConfigureServices`方法中注册“安装”类和模板引擎。
```csharp
services.AddTransient<IConfigureOptions<MvcViewOptions>, PugzorMvcViewOptionsSetup>();
services.AddSingleton<IPugzorViewEngine, PugzorViewEngine>();
```
于是就像魔法一样，我们有了一个可以正常工作的模板引擎。这里有一个演示：
**Controllers/HomeController.cs**
```csharp
public IActionResult Index()
{
    ViewData.Add("Title", "Welcome to Pugzor!");
    ModelState.AddModelError("model", "An error has occurred");
    return View(new { People = A.ListOf<Person>() }); 
}
```
**Views/Home/Index.pug**
```html
block body
	h2 Hello
	p #{ViewData.title} 
	table(class='table')
		thead
			tr
				th Name
				th Title
				th Age
		tbody
			each val in people
				tr
					td= val.firstName
					td= val.title
					td= val.age
```
**输出结果**
```html
<h2>Hello</h2>
<p>Welcome to Pugzor! </p>
<table class="table">
    <thead>
        <tr>
            <th>Name</th>
            <th>Title</th>
            <th>Age</th>
        </tr>
    </thead>
    <tbody>
        <tr><td>Laura</td><td>Mrs.</td><td>38</td></tr>
        <tr><td>Gabriel</td><td>Mr. </td><td>62</td></tr>
        <tr><td>Judi</td><td>Princess</td><td>44</td></tr>
        <tr><td>Isaiah</td><td>Air Marshall</td><td>39</td></tr>
        <tr><td>Amber</td><td>Miss.</td><td>69</td></tr>
        <tr><td>Jeremy</td><td>Master</td><td>92</td></tr>
        <tr><td>Makayla</td><td>Dr.</td><td>15</td></tr>
        <tr><td>Sean</td><td>Mr. </td><td>5</td></tr>
        <tr><td>Lillian</td><td>Mr. </td><td>3</td></tr>
        <tr><td>Brandon</td><td>Doctor</td><td>88</td></tr>
        <tr><td>Joel</td><td>Miss.</td><td>12</td></tr>
        <tr><td>Madeline</td><td>General</td><td>67</td></tr>
        <tr><td>Allison</td><td>Mr. </td><td>21</td></tr>
        <tr><td>Brooke</td><td>Dr.</td><td>27</td></tr>
        <tr><td>Jonathan</td><td>Air Marshall</td><td>63</td></tr>
        <tr><td>Jack</td><td>Mrs.</td><td>7</td></tr>
        <tr><td>Tristan</td><td>Doctor</td><td>46</td></tr>
        <tr><td>Kandra</td><td>Doctor</td><td>47</td></tr>
        <tr><td>Timothy</td><td>Ms.</td><td>83</td></tr>
        <tr><td>Milissa</td><td>Dr.</td><td>68</td></tr>
        <tr><td>Lekisha</td><td>Mrs.</td><td>40</td></tr>
        <tr><td>Connor</td><td>Dr.</td><td>73</td></tr>
        <tr><td>Danielle</td><td>Princess</td><td>27</td></tr>
        <tr><td>Michelle</td><td>Miss.</td><td>22</td></tr>
        <tr><td>Chloe</td><td>Princess</td><td>85</td></tr>
    </tbody>
</table>
```
现在Pug的所有功能都可以正常工作，包括模板继承和内联JavaScript。来我们的[演示站点](https://github.com/AspNetMonsters/pugzor/tree/master/test/pugzore.website)看看例子。

### 打包
所以我们完成了在一天内改变MVC模板引擎这个目标，不过还剩了一点时间，所以我们可以更进一步来创建一个NuGet包。这里有一些挑战，具体涉及在NuGet包中包含所需的node模块。Simon计划为此单独写一篇文章。

你可以自己试一试，添加对`pugzor.core`NuGet包的引用，然后在`Startup.ConfigureServices`方法中的`.AddMvc()`后调用`.AddPugzor()`。
```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Add framework services.
    services.AddMvc().AddPugzor();
}
```
Razor仍会作为默认的模板引擎，但是如果找不到Razor视图文件，MVC框架将会尝试使用PugzorViewEngine，如果能找到相匹配的pug模板，这个模板就会被渲染。
![Pugzor](https://www.davepaquette.com/images/pugzor.png)
我们在这个项目上做了初次尝试，虽然这是一次比较傻的练习，但是以一些有用的东西作收尾。我们真的很惊讶，为MVC创建一个新的模板引擎是这么容易。不过我们不希望Pugzor会受到广泛欢迎，但是既然它能够工作，我们希望把它放在那里并看看人们的想法。

我们有一些问题和一些关于扩展PugzorViewEngine的想法，让我们知道你的想法或是直接来参与贡献代码。我们接受Pull Requests :-)