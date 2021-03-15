---
title: 如何从零创建一个漂亮的 WPF 的 Color Picker 控件
date: 2018-02-01 12:35:06
categories:
 - Tech
tags:
 - WPF
 - Shader
---

在翻Dopamine的issue的时候，发现Dopamine并没有提供一个非常易用的Color Picker来帮助用户创建Accent Color，而这种控件在使用了Material Design的Android App中则非常常见，而且其中多数也非常有漂亮而易用，但是找了一圈并没有发现具有类似设计的WPF控件。本文中需要一点点的初中数学内容，这其中可能有一些错误，或者是算法并非最优，碍于本人水平所限，请见谅。
<!--more-->

如果接触了很多Win32或者WPF程序，例如画图、SAI、Photoshop，会发现其中的拾色器或许很专业，很稳定，但是相对于下图中的控件，则并没有那么好看。微软在UWP SDK 1709中，也加入了这种非常漂亮的拾色器[Color picker](https://docs.microsoft.com/en-us/windows/uwp/design/controls-and-patterns/color-picker)，然而，WPF似乎被忘记了。
![android-color-picker](/content/images/2018/02/android-color-picker.png)

那么如果我需要一个如下图一样漂亮的拾色器该怎么办呢？那凉了呀，只能自己画一个出来了。![digimezzo-wpfcontrols-color-picker](/content/images/2018/02/digimezzo-wpfcontrols-color-picker.png)

那么如何去画这么一个控件呢？最明显的问题在于如何实现那个色轮，从色轮中可以很明显的看出它是由RGB组成的，但好像仅仅用RGB去描述它却有一种说不清的障碍，三个[0,255]的值为什么可以组成一个360°的圆？因为使用RGB分量的描述并不符合我们的感官直觉（尽管视锥细胞感受到的依然是RGB），因此，这里需要将RGB空间转换为[HSV空间](https://en.wikipedia.org/wiki/HSL_and_HSV)来描述同样一个颜色。

那么，我们现在已经有了数学模型，可是如何将其显示在一个Ellipse上呢？应该遍历填充一个Brush吗？不不不，这太不优雅了，当然是要使用Shader把色轮交给GPU来渲染，这样一来你不仅可以轻易得到60帧的UI，而且甚至可以吹一吹硬件加速（逃。有关Shader的前置知识请移步[在 WPF 中使用 Pixel Shader 为控件添加特效](https://blog.magentaize.net/add-effect-to-controls-with-pixel-shader-in-wpf/)。

## 编写Shader
>这部分代码使用了JohanLarsson所写的[Gu.Wpf.Geometry](https://github.com/JohanLarsson/Gu.Wpf.Geometry)，根据MIT协议发布

接下来便要开始编写一个Shader来实现色轮，显然我们需要先用几个常量寄存器来存放一些外部参数，顺带写上几个常量：
```glsl
static float PI = 3.14159274f;
static float PI2 = 6.28318548f;
// 圆心坐标
static float2 cp = float2(0.5, 0.5);

// 内半径
float InnerRadius : register(C0);
// 中心饱和度
float InnerSaturation : register(C1);
// 明度
float Value : register(C2);
// 色相起始角
// 普遍情况下平面欧几里得空间的一个角度在笛卡尔坐标系中是由指向X正半轴的射线旋转而来，此时起始角为0°，
// 但有时，也就是在许多色轮的实现中，起始角为90°
float StartAngle : register(C3);
```

虽然说我们是用了Shader来渲染，但实际上的算法还是逃不出用遍历改变每像素颜色填充Brush的套路（笑。那么整个Shader的逻辑可以分为两部分，区域内的像素->着色，区域外的像素->透明，而这个区域则是一个圆环（默认宽度为0）：
```glsl
// code by http://www.chilliant.com/rgb2hsv.html
float3 HUEtoRGB(in float H)
{
    float R = abs(H * 6 - 3) - 1;
    float G = 2 - abs(H * 6 - 2);
    float B = 2 - abs(H * 6 - 4);
    return saturate(float3(R, G, B));
}

float3 HSVtoRGB(in float3 HSV)
{
    float3 RGB = HUEtoRGB(HSV.x);
    return ((RGB - 1) * HSV.y + 1) * HSV.z;
}

float4 main(float2 uv : TEXCOORD) : COLOR
{
    float2 rv = uv - cp;
    // 像素到圆心的欧几里得距离
    float r = length(rv);
    float ir = InnerRadius / 2;
    // 像素是否落在圆环内
    if (r >= ir && r <= 0.5)
    {
        // 计算当前像素的HSV
        // float h = xxx, s = xxx, v = xxx;
        return float4(HSVtoRGB(float3(h, s, v)), 1);
    }
    return float4(0, 0, 0, 0);
}
```
那么接下来只需要计算出当前像素应具有的(h, s, v)三元组即可。其中h的值均匀分布在θ = [0, 2π]的区间内，s的值均匀分布在r = [InnerRadius, OutterRadius - InnerRadius]的区间内，而v直接使用寄存器的值即可。

不过！为方便描述，以上的区间都是极坐标，在实际渲染的时候需要将极坐标再转换为归一化笛卡尔坐标以供GPU处理。那么实际的逻辑应表达为：
```glsl
// 线性插值
float interpolate(float min, float max, float value)
{
    if (min == max)
    {
        return 0.5;
    }

    if (min < max)
    {
        return clamp((value - min) / (max - min), 0, 1);
    }

    return clamp((value - max) / (min - max), 0, 1);
}

float clamp_angle_positive(float a)
{
    if (a < 0)
    {
        return a + PI2;
    }

    return a;
}

float clamp_angle_negative(float a)
{
    if (a > 0)
    {
        return a - PI2;
    }

    return a;
}

float angle_from_start(float2 uv, float2 center_point, float start_angle, float central_angle)
{
    float2 v = uv - center_point;
    return central_angle > 0
        ? clamp_angle_positive(clamp_angle_positive(atan2(v.x, -v.y)) - clamp_angle_positive(start_angle))
        : abs(clamp_angle_negative(clamp_angle_negative(atan2(v.x, -v.y)) - clamp_angle_negative(start_angle)));
}

float4 main(float2 uv : TEXCOORD) : COLOR
{
    // some codes...
    if (r >= ir && r <= 0.5)
    {
        // 计算当前像素的HSV
        // 将色相起始角转换为弧度
        float sa = radians(StartAngle);
        // 根据色相起始角的偏移旋转后的对应归一化坐标系的色相
        float h = interpolate(0, PI2, angle_from_start(uv, cp, sa, PI2));
        float s = lerp(InnerSaturation, 1, interpolate(ir, 0.5, r));
        float v = Value;
    }
    return float4(0, 0, 0, 0);
}
```

对应的C#包装器则没有什么好特别说明的，注意几个寄存器的初始值即可，代码请移步：[HsvWheelEffect.cs](https://github.com/Magentaize/WPFControls/blob/d49a498f629c023c5ff241eaac075fca7a4eef4c/Digimezzo.WPFControls/Effects/HsvWheelEffect.cs)

## 编写布局
从设计图中可以看出，整个控件应分为三部分，也就是色轮和预览框、滑动条、数值框，每部分都可以用StackPanel包裹起来然后将三个StackPanel放到最外层的StackPanel中并垂直排列，而色轮和预览这部分可以用Canvas容器直接进行绝对布局，并给色轮加一个上文中完成的ShaderEffect，数值框则需要用Grid容器并划分为四行两列。同时你会注意到，色轮上还有一个黑色的Ellipse作为拾色器位置指示器，这是实际上一个Thumb控件，使用Canvas的另一个好处是易于定位Thumb。

因此，抛开其他不是那么重要的样式，布局的问题基本就是一个大概的这种样子，各个控件的属性再绑定到逻辑代码上即可。最终的代码请移步：[ColorPicker.xaml](https://github.com/Magentaize/WPFControls/blob/d49a498f629c023c5ff241eaac075fca7a4eef4c/Digimezzo.WPFControls/Themes/ColorPicker.xaml)

不过需要注意HEX和颜色分量的TextBox需要对输入值作判断，也就是在绑定里添加一个`Binding.ValidationRules`集合，将RGB限制在[0, 255]内，HSV限制在[0, 360]和[0, 100]内，HEX限制在“^#[0-9a-fA-F]{6}”的正则条件内。这需要增加两个数据验证器，以HEX的验证为例：
```xml
<TextBox Width="70">
 <TextBox.Text>
  <Binding Path="Hex"
   UpdateSourceTrigger="PropertyChanged"
   RelativeSource="{RelativeSource TemplatedParent}">
    <Binding.ValidationRules>
     <validations:RegexValidation Pattern="^#[0-9a-fA-F]{6}"/>
    </Binding.ValidationRules>
  </Binding>
 </TextBox.Text>
</TextBox>
```

```csharp
using System.Globalization;
using System.Text.RegularExpressions;
using System.Windows.Controls;

namespace Digimezzo.WPFControls.ValidationRules
{
    public class RegexValidation : ValidationRule
    {
        public string Pattern
        {
            get => _pattern;
            set
            {
                _pattern = value;
                _isRegexChanged = true;
            }
        }

        private string _pattern;
        private bool _isRegexChanged = false;
        private Regex _regex;

        public override ValidationResult Validate(object value, CultureInfo cultureInfo)
        {
            // 仅当匹配模式改变时才重新编译表达式以增强性能
            if (_isRegexChanged)
            {
                _regex = new Regex(_pattern);
                _isRegexChanged = false;
            }

            return new ValidationResult(_regex.IsMatch((string) value), value);
        }
    }
}
```

## 编写逻辑
逻辑部分抛开依赖属性与一些控件字段，主要的复杂度在于协调各个控件之间的绑定属性。因为一个作为一个拾色器，其几乎所有的控件都最终绑定到了同一个Color属性上，而修改该属性会触发PropertyChanged事件，同时在修改任意一个控件的属性值时，我们希望该控件的属性值不变而其他的控件属性值会自动变化，这就陷入了一个循环调用的境地中。因此，我们需要剥离出最终改变属性的逻辑，并将该逻辑认定为一个原子操作。

于是，将所有依赖属性的回调方法设置为同一个方法，在该方法中，当一个请求被处理时，将屏蔽所有其他请求（虽然在我的姿势范围内应该这样写，但是我觉得这样效率并不高，也不优雅，希望能指出更优雅的方案）。

但是这还有一个问题，所有依赖属性及其相关的字段、方法都需要是静态的，但是对于每一个实例的内部状态是不一样的，需要强行调用一下实例方法。
```csharp
private bool withinColorChange = false;

public static readonly DependencyProperty HueProperty = DependencyProperty.Register(nameof(Hue), typeof(double), typeof(ColorPicker), new PropertyMetadata(0d, ComponentChangedCallback));

private static void ComponentChangedCallback(DependencyObject d, DependencyPropertyChangedEventArgs e) => ((ColorPicker)d).OnComponentChanged(e);

private void OnComponentChanged(DependencyPropertyChangedEventArgs e)
{
    // 临界区互斥条件
    if (withinColorChange)
        return;
    // 进入临界区
    withinColorChange = true;

    // 响应唯一一个属性的变化
    if (e.Property == HexProperty)
    {
        var rgb = HexToRgb((string)e.NewValue);
        UpdateHsv(rgb);
    }
    else if (e.Property == RedProperty)
    {
        var rgb = Color.FromArgb(255, Convert.ToByte(e.NewValue), SelectedColor.G, SelectedColor.B);
        UpdateHsv(rgb);
    }
        else if (e.Property == GreenProperty)
    {
        var rgb = Color.FromArgb(255, SelectedColor.R, Convert.ToByte(e.NewValue), SelectedColor.B);
        UpdateHsv(rgb);
    }
    else if (e.Property == BlueProperty)
    {
        var rgb = Color.FromArgb(255, SelectedColor.R, SelectedColor.G, Convert.ToByte(e.NewValue));
        UpdateHsv(rgb);
    }
    else if (e.Property == SelectedColorProperty)
    {
        UpdateAllComponents();
    }
    else
    {
        var hsv = new Hsv(Hue, Saturation, Value);
        UpdateRgb(hsv);
        // Value改变时不改变Silder背景色，而只改变属性值
        if (e.Property != ValueProperty)
            UpdateValueSlider(hsv);
    }

    ResetPickerThumbPosition();

    // 退出临界区
    withinColorChange = false;
}

private void UpdateAllComponents()
{
    var hsv = RgbToHsv(SelectedColor);
    Hue = hsv.Hue;
    Saturation = hsv.Saturation;
    Value = hsv.Value;
    Red = SelectedColor.R;
    Green = SelectedColor.G;
    Blue = SelectedColor.B;
    Hex = RgbToHex(SelectedColor);

    UpdateValueSlider(hsv);
}

private void UpdateHsv(Color rgb)
{
    var hsv = RgbToHsv(rgb);
    Hue = hsv.Hue;
    Saturation = hsv.Saturation;
    Value = hsv.Value;
    SelectedColor = rgb;
    Hex = RgbToHex(rgb);

    UpdateValueSlider(hsv);
}

private void UpdateRgb(Hsv hsv)
{
    var rgb = HsvToRgb(hsv);
    Red = SelectedColor.R;
    Green = SelectedColor.G;
    Blue = SelectedColor.B;
    SelectedColor = rgb;
    Hex = RgbToHex(rgb); 
}

private void UpdateValueSlider(Hsv hsv)
{
    hsv.Value = 1.0;
    var rgb = HsvToRgb(hsv);
    SelectedFullValueColor = rgb;
}
```
而仅仅更新了所有属性的值功能并没有完成，色轮上方的Thumb控件还应该既能跟随鼠标移动位置，还能在颜色分量变化时移动到色轮的正确位置处。
而Thumb位置坐标的计算则需要根据HSV值来完成，也就是将Shader中的极坐标->笛卡尔坐标用C#重新实现一个，不过在逻辑代码中需要考虑的条件更多，因为其空间不是归一化的，所有的距离都需要再乘以一个半径的比率，如下：
```csharp
private void ResetPickerThumbPosition()
{
    var radian = AngleToRadian(Hue * 360d);
    double nX = Math.Cos(radian) * Saturation * Radius, nY = Math.Sin(radian) * Saturation * Radius;
    Canvas.SetLeft(pThumb, 150 - nX);
    Canvas.SetTop(pThumb, 150 - nY);
}
```
当用鼠标拖动Thumb到一个位置时，需要将其在Canvas容器内的笛卡尔坐标转换为极坐标，根据极坐标计算出精确的HSV值再转换成RGB值，同时根据临界条件防止Thumb被拖出色轮的范围之外，如下：
```csharp
private void GetColorFromCurrentPositionAndMoveThumb(double nX, double nY)
{
    double diffX = nX - Radius, diffY = nY - Radius;
    var radian = Math.Atan(diffY / diffX);
    var hue = RadianToAngle(radian);
    if ((nX >= Radius && nY < Radius) || (nX >= Radius && nY >= Radius))
        hue = hue + 180;
    if (nX < Radius && nY >= Radius)
        hue = hue + 360;
    Hue = hue / 360d;

    var euclid = diffX * diffX + diffY * diffY;
    if (Radius2 - euclid >= 0)
    {
        Canvas.SetLeft(pThumb, nX);
        Canvas.SetTop(pThumb, nY);
        Saturation = Math.Sqrt(euclid) / 150d;
    }
    else
    {
        Saturation = 1d;
        nX = Math.Cos(radian) * Radius;
        nY = Math.Sin(radian) * Radius;
        if (diffX < 0)
        {
            Canvas.SetLeft(pThumb, Radius - nX);
            Canvas.SetTop(pThumb, Radius - nY);
        }
        else
        {
            Canvas.SetLeft(pThumb, Radius + nX);
            Canvas.SetTop(pThumb, Radius + nY);
        }
    }

    var hsv = new Hsv(Hue, Saturation, Value);
    UpdateRgb(hsv);
    UpdateValueSlider(hsv);
}
```
关于上文中用到的HSV与RGB相互转换的算法：
>这部分代码使用了objorke所写的[PropertyTools](https://github.com/objorke/PropertyTools)，根据MIT协议发布
