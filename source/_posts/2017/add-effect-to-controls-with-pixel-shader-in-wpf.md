---
title: 在 WPF 中使用 Pixel Shader 为控件添加特效
date: 2017-09-10 15:16:22
categories:
 - Tech
tags: 
- Shader
- WPF
---

WPF中内置了许多中特效与变换，在一般的日常使用中已经足够，但是如果想要实现更加强大的特效，必须使用HLSL(High Level Shader Language/高级着色器语言)来手写Pixel Shader(像素着色器)以便能够为控件带来丰富多彩的效果，比如说下图中的边缘羽化特效：
<!--more-->
<video style="height:100%; width:100%" src="/content/images/2017/09/feathering.webm" controls="controls">
Your browser does not support HTML5 video.
</video>

我们知道，WPF中内置的特效都是针对整个控件而生效的，那么这种能够针对部分区域生效的特效是如何编写的呢？实际上，这是将整个控件放在GPU中渲染出来的，而并非在C#或XAML代码中实现的。那么显然你会有一个疑问，不用C#怎么来实现特效？WPF不是使用C#来描述逻辑的吗？
特效不在C#和XAML中实现并不代表我们不去书写C#代码，而是关键的部分不使用C#来写，C#仅仅作为一个包装器(wrapper)来提供GPU到WPF的通信桥梁。

说了一堆听不懂的话，那么究竟如何来写这个叫做Pixel Shader的东西呢？别急，先准备一下环境和相关的基本概念。

## 预备工作
### 基本概念
1.着色器(Shader)
这是一个令人容易误解的概念。着色器并不是字面意思上的一个内置于GPU中的物理器件，就像CPU中的加法器、乘法器那样，相反，着色器是一段程序，也有输入和输出，GPU在流处理器中运行着色器这段程序，经过某种算法将输入处理为输出，GPU会获取到着色器的输出并显示出来。

2.高级着色器语言(HLSL)
着色器是一段程序，是二进制的，而HLSL则是编写着色器的一种语言，是文本可读的。

3.渲染管线(Rendering Pipeline)
这又是一个令人容易误解的概念，渲染管线并不是一种管道或是一种线路，反而是和流水线一样，是一个逻辑上的概念，意味着在GPU中执行的一系列操作和步骤。

4.顶点着色器(Vertex Shader)
无论你的图像和模型有多么复杂，GPU都只能处理三角形，于是，一个模型实际上是由许多三角形组成的(曲面细分233)，每个三角形有三个角，每个角的拐点的坐标就叫做顶点，而处理顶点的程序叫做顶点着色器。它是渲染管线的一部分。
![model-triangles](/content/images/2018/02/model-triangles.gif)<center>三角形生成的模型</center>

5.像素着色器(Pixel Shader)
像素着色器是在DirectX中的名称，在OpenGL中又称为片断着色器(Fragment Shader)。经过顶点着色器处理后的数据，会被送到像素着色器中，但是这个数据并没有任何可视化的数据，比如，没有边框宽度，没有内部填充颜色，没有光照，也就是说，这个图像是完全透明的。而经过像素着色器，将通过一个算法，为输入的图像上色，这样我们才能看到。它是渲染管线的一部分。
![shader-and-pipeline](/content/images/2018/02/shader-and-pipeline.png)<center>渲染管线和着色器</center>

6.纹理(Texture)
在3D中纹理是给几何素材上色的素材，在2D中则是填充平面图形的图像。

### 准备环境
1.Directx SDK
想要对GPU编程，首先你需要安装DirectX SDK，这里对版本的需求不高，9.0c的版本就足以够用，如果你正在使用Visual Studio 2017，在安装时勾选DirectX游戏开发即可自动安装。

2.HLSL Tools for Visual Studio
Visual Studio默认并不提供对HLSL的语法高亮支持，若有需要，你可以安装[HLSL Tools for Visual Studio](https://marketplace.visualstudio.com/items?itemName=TimGJones.HLSLToolsforVisualStudio)这个插件来提供语法高亮支持。

## 编写着色器
### 编写fx文件
*<center>由于WPF仅支持到Shader Model 3.0，因此本文也是使用SM3.0的语法来编写。</center>*

现在，我们将开始用HLSL来编写着色器，然而，HLSL是一门全新的语言，每一行代码我都会详细解释，不过也不用担心，因为HLSL的语法风格和C非常相似，你会很快上手的。

先上代码：
```glsl
float implicitfeather : register(c0);
float implicitwidth : register(c1);
float implicitheight : register(c2);
sampler2D implicitInputBackground : register(s0);

float4 main(float2 uv : TEXCOORD0) : COLOR0
{
    float4 colorSample = tex2D(implicitInputBackground, uv);
    float width = uv.x * implicitwidth;
    float feather = implicitfeather;
    if (width < feather)
    {
        colorSample *= width / feather;
    }
    if (width > implicitwidth - feather)
    {
        colorSample *= (implicitwidth - width) / feather;
    }

    return colorSample;
}
```

你会发现HLSL的语法乍看上去非常熟悉，浓重的面向过程C系风格，但是关键字的定义却大不相同，以下我将逐行解释。

```glsl
float implicitfeather : register(c0);
float implicitwidth : register(c1);
float implicitheight : register(c2);
```
首先是前三行定义了三个全局变量，分别是*羽化半径*、*纹理宽度*、*纹理高度*，并且都是float类型的，但是却在变量名后面加了一个冒号和其他东西，这叫做`语义(semantic)`，语义并不定义了变量的类型，而是定义了变量在输入输出过程中的含义，也就是在渲染管线中扮演的角色。当一个顶点在顶点着色器中处理完毕之后，将会被送往像素着色器中，但是顶点着色器和像素着色器并不在同一个上下文环境中，它们之间无法像函数调用一样直接传递参数，因此，定义了语义这个东西来提供多个着色器之间的数据交换，对于顶点着色器的输出变量和像素着色器的输入变量，如果声明的语义相同，那么就会访问到相同的变量。

`register`就是一个语义，代表着这是一个在寄存器中的变量，而后面的括号内的内容则表示了实际上是使用了哪一个寄存器，此处的`c[n]`代表这是第n个常量寄存器，至于我们如何把变量传递到着色器的寄存器中，将在下文的C#代码部分说明，此时就当做该变量已经被赋值即可。
对于相关的寄存器类型的说明，可以访问此处[ps_3_0 Registers](https://msdn.microsoft.com/en-us/library/windows/desktop/bb172920(v=vs.85).aspx)。

接下来又定义了一个全局变量，是*输入纹理*：
```glsl
sampler2D implicitInputBackground : register(s0);
```
`sampler2D`是一个未曾见过的变量类型，它是一个与纹理相对应的内置数据类型，叫做`采样器(sampler)`，而后面的2D则代表这是一个2D的纹理，并且保存在采样器寄存器中。

```glsl
float4 main(float2 uv : TEXCOORD0) : COLOR0 {}
```
这是像素着色器的入口函数，函数签名由*返回值类型*、*函数名*、*参数列表*、*返回值语义*组成，一个着色器中可以有多个函数，但非入口函数不必声明语义，并且入口函数的默认值为`main`，可以在编译时指定为其他函数。

`float[n]`也是一种新的变量类型，代表着由n个float一起组成的一个变量，因为像素着色器返回的结果是一个像素，而一个像素由四个值来定义，分别是ARGB，因此主函数的返回值是float4。

那么uv又是什么？实际上像素着色器的输入是由顶点着色器输出的数据，并且不止是一个值，完整的写法应该是这样的：
```glsl
struct VS_OUTPUT
{
   float4 Position  : SV_POSITION;
   float4 Color     : COLOR0;
   float2 UV        : TEXCOORD0;
};

float4 main(VS_OUTPUT input) : COLOR0 {}
```
显然，像素着色器是有三个输入的，它们被包含在一个名为`VS_OUTPUT`的结构中，而VS即是Vertex Shader的缩写，那么这三个变量又是什么？Position的语义是`SV_POSITION`，代表了当前将被渲染的像素在View Space上的坐标(不是屏幕的坐标)，然而由于在WPF中我们不能手动设置View Space；Color的语义是`COLOR0`，代表了顶点中储存的第一个颜色；UV的语义是`TEXCOORD0`，代表了纹理的第一个坐标，而之所以坐标的名字是UV而不是XY，因为XY表示了像素的坐标，大家也就都这么用了。既然有第一个颜色和第一个坐标，那么就有第二个颜色和第二个坐标，但是此处并不需要。

那么又有一个问题，为什么入口函数的参数既可以是一个变量又可以是一个结构体呢？因为当入口函数被调用时，并不是和x86汇编一样按照调用协定(\_\_stdcall什么的)将函数压栈，而是根据语义来为变量赋值，你甚至可以把主函数写成下面这样，同样可以正确运行：
```glsl
float4 main(float2 UV : TEXCOORD0, float4 Position : SV_POSITION, float4 Color : COLOR0) : COLOR0 {}
```

```glsl
float4 colorSample = tex2D(implicitInputBackground, uv);
float width = uv.x * implicitwidth;
float feather = implicitfeather;
```
接下来我们定义了三个变量，`tex2D(s, t)`是一个函数，用来对一个采样器中的一个点进行采样，这样能得到采样器中坐标为uv的这个点的像素，然后再使用uv的x轴坐标去乘以纹理宽度就能得到当前像素在纹理中的横坐标。第三个变量则是直接取得了寄存器中的值。

此处为什么这样做就能得到纹理中的横坐标需要解释一下。对于一个纹理，我们使用的是像素坐标系，但是着色器使用的是归一化后的坐标系，在这两个坐标系之间我们需要进行一个逆变换才能得到正确的值。
![pixel-coordinates-and-normalized-coordinates](/content/images/2018/08/pixel-coordinates-and-normalized-coordinates.png)<center>像素坐标系和归一化坐标系</center>

```glsl
if (width < feather)
{
    colorSample *= width / feather;
}
if (width > implicitwidth - feather)
{
    colorSample *= (implicitwidth - width) / feather;
}
```
这部分是一个简单的线性羽化算法，将给定的每像素按照与羽化半径的距离进行线性变换，ARGB的范围是[0.0f, 1.0f]的闭区间，对应的颜色就是从完全透明到黑色。

你可能又要问，这不是只处理了一个像素吗，那怎么能让控件的左右两边刚好羽化而中间不变呢？对于只接触过CPU编程的人而言，这里确实难以理解，这段代码既没有对像素的遍历，又没有像素的集合，为什么突然就实现了羽化呢？原因是，对于GPU编程而言，一段代码在微观上是串行执行的，但是在宏观上是并行执行的，GPU拥有多个流处理器，每个流处理器将执行对一个像素的运算，但是同时会有多个流处理器去计算许多个像素，也就是说，你给GPU分配了一个计算图像的任务，并不是由单独一个流处理器用一个大循环来独自执行整个任务，而是会把这一个图像拆成多个像素，同时执行，这也就意味着，在同一时刻，有多个流处理器在同时执行编写的像素着色器，对于每个流处理器而言，其他流处理器在执行什么它是不知道的。

```glsl
return colorSample;
```
最后这一行就很简单了，将计算后的像素返回即可。

### 编译
和其他代码一样，HLSL写的程序也需要编译，不过用到的编译器并不是MSBuild而是fxc。我们将上面写的shader保存为FeatheringEffect.fx，在控制台中输入：
```html
"C:\Program Files (x86)\Microsoft DirectX SDK (June 2010)\Utilities\bin\x86\fxc.exe" FeatheringEffect.fx /T ps_3_0 /Fo FeatheringEffect.ps
```
其中`/T ps_3_0`表示编译为Shader Model 3.0的Pixel Shader文件，`/Fo FeatheringEffect.ps`表示输出的文件路径。如果编译成功，则会显示
*compilation succeeded;*

接下来，我们就可以进行C#的包装了。

## 使用C#加载着色器
在WPF中，所有自定义特效必须从`ShaderEffect`继承，因此，我们先建立一个类：
```csharp
public class FeatheringEffect : ShaderEffect {}
```
然后添加三个依赖属性，这三个依赖属性将完成变量XAML->C#->Shader的传递，并与在Shader中声明的全局变量*羽化半径*、*纹理宽度*、*纹理高度*相对应。
```csharp
#region Properties
public double FeatheringRadius
{
    get => (double) GetValue(FeatheringRadiusProperty);
    set => SetValue(FeatheringRadiusProperty, value);
}

public double TexWidth
{
    get => (double)GetValue(TexWidthProperty);
    set => SetValue(TexWidthProperty, value);
}
#endregion

#region Dependency Properties
public static DependencyProperty InputBackgroundProperty = RegisterPixelShaderSamplerProperty("InputBackground", typeof(FeatheringEffect), 0);

public static DependencyProperty FeatheringRadiusProperty = DependencyProperty.Register("FeatheringRadius", typeof(double), typeof(FeatheringEffect), new UIPropertyMetadata(default(double), PixelShaderConstantCallback(0)));

public static DependencyProperty TexWidthProperty = DependencyProperty.Register("TexWidth", typeof(double), typeof(FeatheringEffect), new UIPropertyMetadata(default(double), PixelShaderConstantCallback(1)));
#endregion
```
你可能注意到了，这三个依赖属性的定义方式似乎和控件中的不太一样，因为这是要传递给着色器的变量，而并非传递给WPF图形框架的。
```csharp
RegisterPixelShaderSamplerProperty("InputBackground", typeof(FeatheringEffect), 0)
```
这个方法的前两个参数没什么可说的，第三个参数指定了在着色器中相关联的是第几个纹理寄存器，也就是下面的`s0`。
```glsl
sampler2D implicitInputBackground : register(s0);
```

```csharp
DependencyProperty.Register("FeatheringRadius", typeof(double), typeof(FeatheringEffect), new UIPropertyMetadata(default(double), PixelShaderConstantCallback(0)))
```
这个方法的第四个参数需要一个`UIPropertyMetadata`对象以能够与着色器中的常量寄存器相关联，而其构造函数的第二个参数就是指定为第几个常量寄存器。

```csharp
public FeatheringEffect()
{
    PixelShader = new PixelShader()
    {
        UriSource = UriUtils.MakePackUri<FeatheringEffect>("Presentation/Effects/FeatheringEffect.ps")
    };;
    UpdateShaderValue(InputBackgroundProperty);
    UpdateShaderValue(FeatheringRadiusProperty);
    UpdateShaderValue(TexWidthProperty);
}
```
在构造函数中，我们需要给`PixelShader`实例化一个对象，并且将其`UriSource`属性赋值为编译的像素着色器的Uri，然后调用三次`UpdateShaderValue(dp)`方法将三个依赖属性与着色器建立关联。你又会发现，此处没有为`InputBackgroundProperty`建立对应的包装属性，因为该依赖属性不需要由我们手动赋值，为其设置了纹理寄存器后WPF将自动把当前控件的`Visual`转换成`Brush`赋值给寄存器。

与其他内置特效一样，只要在XAML中添加特效即可应用：
```xml
<Control.Effects>
    <FeatheringEffect FeatheringRadius="20" TexWidth={Binding ActualWidth}/>
</Control.Effects>
```
这样一来仅仅用了几kb就完成了强大的特效，是不是美滋滋。

相关的源码：
[Github](https://github.com/Magentaize/Dopamine/tree/51414eb2c14e03eee8165cb03d0d1e4ef082da89/Dopamine.Common/Presentation/Effects)

<font color='grey'>图片来自DirectXTutorial.com</font>