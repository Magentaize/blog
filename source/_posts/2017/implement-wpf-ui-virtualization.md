---
title: 正确实现WPF中的UI虚拟化
date: 2017-06-04 09:31:20
categories:
 - Tech
tags: 
 - WPF
---

WPF中内置实现的虚拟化容器只有`VirtualizingStackPanel`这一个，而为了应对实际应用中的各种自定义（组合）控件，我们需要自己去实现容器应该完成的功能，若想做到这一点，不仅仅需要继承`VirtualizingPanel`，还需要实现`IScrollInfo`接口。
一般来说，我们很少会去实现`IScrollInfo`这个接口，因为这个接口实在是过于繁重，定义了多达 15 个方法和 9 个属性！但是为了能够正确地在`ScrollViewer`中处理各种情况，实现这个接口是必须的。
<!--more-->

先把将要实现的这个面板命名为`VirtualizingWrapPanel`，添加一个`TranslateTransform`字段，这是为了完成`IScrollInfo`的实现，使之能够正确地表现出上下滚动的效果。
```csharp
public class VirtualizingWrapPanel : VirtualizingPanel, IScrollInfo
{
    private TranslateTransform trans = new TranslateTransform();
}
```
在构造函数中把trans赋值给相应的字段。
```csharp
public VirtualizingWrapPanel()
{
    this.RenderTransform = trans;
}
```
作为一个控件，就需要有长和宽这两个属性，而又同时作为一个容器，还需要能够设置容器内的子控件的长和宽。
```csharp
public static readonly DependencyProperty ChildWidthProperty = DependencyProperty.RegisterAttached("ChildWidth", typeof(double), typeof(VirtualizingWrapPanel), new FrameworkPropertyMetadata(200.0, FrameworkPropertyMetadataOptions.AffectsMeasure | FrameworkPropertyMetadataOptions.AffectsArrange));

public double ChildWidth
{
    get => Convert.ToDouble(GetValue(ChildWidthProperty));
    set => SetValue(ChildWidthProperty, value);
}

public static readonly DependencyProperty ChildHeightProperty = DependencyProperty.RegisterAttached("ChildHeight", typeof(double), typeof(VirtualizingWrapPanel), new FrameworkPropertyMetadata(200.0, FrameworkPropertyMetadataOptions.AffectsMeasure | FrameworkPropertyMetadataOptions.AffectsArrange));

public double ChildHeight
{
    get => Convert.ToDouble(GetValue(ChildHeightProperty));
    set => SetValue(ChildHeightProperty, value);
}
```
当容器被创建、改变大小的时候，容器需要做到两件事，分别是“测量”和“布局”，这对应两个函数`MeasureOverride(Size)`和`ArrangeOverride(Size)`，而UI虚拟化，也就是在这两个步骤中，只去测量与布局实际会显示在屏幕上的那部分控件，对于不会显示出来的控件则不去对其实例化。

根据MSDN所写，每当附加到`ScrollViewer`上的`IScrollInfo`其表示滚动的属性（scrolling property）`offset`、`extent`、`viewport`这三个发生变化时，应执行`ScrollViewer.InvalidateScrollInfo()`方法，那么先来实现这个部分。

实现`IScrollInfo`所声明的属性
```csharp
public ScrollViewer ScrollOwner { get; set; }

public bool CanHorizontallyScroll { get; set; }

public bool CanVerticallyScroll { get; set; }

private Point offset;
public double HorizontalOffset => this.offset.X;
public double VerticalOffset => this.offset.Y;

private Size extent = new Size(0, 0);
public double ExtentHeight => this.extent.Height;
public double ExtentWidth => this.extent.Width;

private Size viewport = new Size(0, 0);
public double ViewportHeight => this.extent.Height;
public double ViewportWidth => this.extent.Width;
```
实现需要执行`InvalidateScrollInfo()`的逻辑。
```csharp
private void UpdateScrollInfo(Size availableSize)
{
    var extent = CalculateExtent(availableSize, GetItemCount(this));

    if (extent != this.extent)
    {
        this.extent = extent;
        this.ScrollOwner.InvalidateScrollInfo();
    }

     if (availableSize != this.viewport)
     {
        this.viewport = availableSize;
        this.ScrollOwner.InvalidateScrollInfo();
     }
}
```
为了得到数据源中的子项数量，需要先得到拥有数据源的控件，而这类控件都继承于`ItemsControl`。
```csharp
private int GetItemCount(DependencyObject element)
{
    var itemsControl = ItemsControl.GetItemsOwner(element);
    int itemCount = itemsControl.HasItems ? itemsControl.Items.Count : 0;

    return itemCount;
}
```
为了能够得到`extent`的大小，需要根据容器尺寸与容器内子项的数量来确定，由于容器并不是单方向的，而是同时有行和列，所以还需要计算出每行中的子项数量。
```csharp
private Size CalculateExtent(Size availableSize, int itemCount)
{
    int childrenPerRow = CalculateChildrenPerRow(availableSize);

    return new Size(childrenPerRow * this.ChildWidth, this.ChildHeight * Math.Ceiling(Convert.ToDouble(itemCount) / childrenPerRow));

private int CalculateChildrenPerRow(Size availableSize)
{
    int childrenPerRow = 0;

    if (availableSize.Width == double.PositiveInfinity)
        childrenPerRow = this.Children.Count;
    else
        childrenPerRow = Math.Max(1, Convert.ToInt32(Math.Floor(availableSize.Width / this.ChildWidth)));

    return childrenPerRow;
```

接下来便是要实现“测量”`MeasureOverride(Size)`，首先需要确定能够显示的第一个子项与最后一个子项在数据源中的索引。
```csharp
private void GetVisibleRange(ref int firstVisibleItemIndex, ref int lastVisibleItemIndex)
{
    int childrenPerRow = CalculateChildrenPerRow(this.extent);

    firstVisibleItemIndex = Convert.ToInt32(Math.Floor(this.offset.Y / this.ChildHeight)) * childrenPerRow;
    lastVisibleItemIndex = Convert.ToInt32(Math.Ceiling((this.offset.Y + this.viewport.Height) / this.ChildHeight)) * childrenPerRow - 1;

    // 如果可显示的最后一项的是数据源的最后一个，那么最后一个子项的索引应该是数据源最后一个元素的的索引
    if (lastVisibleItemIndex >= GetItemCount(this))
        lastVisibleItemIndex = itemCount - 1;
}
```
接下来需要使用`IItemContainerGenerator`来控制（生成、添加、删除）实际会显示出的每一个子项，所有的这些可显示子项都保存在`UIElementCollection`集合中。
```csharp
// 由于一个bug，必须先访问InternalChildren，否则ItemContainerGenerator会返回null
var children = this.InternalChildren;
var generator = this.ItemContainerGenerator;
```
获得generator中第一个可显示子项的位置`GeneratorPosition`。
```csharp
var startPos = generator.GeneratorPositionFromIndex(firstVisibleItemIndex);
```
在继续写之前，要讲一下这个`GeneratorPosition`，它有两个属性，其中`Offset`表示firstVisibleItemIndex与上一个已经添加进`UIElementCollection`集合的子项在数据源中的索引的偏移，如果firstVisibleItemIndex已经被添加进集合，其值为0，否则从1开始计数；`Index`是firstVisibleItemIndex在`UIElementCollection`集合中的索引，如果firstVisibleItemIndex是虚拟的（未被添加进集合），那么其值为上一个已经添加进集合的子项的索引。因此，在添加（可视化）第一个子项前，需要确定这一项的上一项是否已经可视化。
```csharp
int childIndex = (startPos.Offset == 0) ? startPos.Index : startPos.Index + 1;
```
现在就可以将可以被显示出的子项添加进`UIElementCollection`中。
```csharp
using (generator.StartAt(startPos, GeneratorDirection.Forward, true))
{
    int itemIndex = firstVisibleItemIndex;
    while (itemIndex <= lastVisibleItemIndex)
    {
        bool newlyRealized = false;

        // 生成UI元素
        var child = generator.GenerateNext(out newlyRealized) as UIElement;
        if (newlyRealized)
        {
            // 如果是下滚，需要向后添加子项，如果是上滚，则需要向前添加子项
            if (childIndex >= children.Count)
                base.AddInternalChild(child);
            else
                base.InsertInternalChild(childIndex, child);
            generator.PrepareItemContainer(child);
        }

    // 为UI元素提供测量尺寸
    child.Measure(new Size(this.ChildWidth, this.ChildHeight));
    itemIndex += 1;
    childIndex += 1;
}
```
UI虚拟化正是为了提高性能，在向集合中添加可显示的子项之后，需要将其他不在可显示范围内的子项移除。
```csharp
private void CleanUpItems(int firstVisibleItemIndex, int lastVisibleItemIndex)
{
    var children = this.InternalChildren;
    generator = this.ItemContainerGenerator;

    for (int i = children.Count - 1; i >= 0; i += -1)
    {
        childGeneratorPos = new GeneratorPosition(i, 0);
        int itemIndex = generator.IndexFromGeneratorPosition(childGeneratorPos);
        if (itemIndex < firstVisibleItemIndex || itemIndex > lastVisibleItemIndex)
        {
            generator.Remove(childGeneratorPos, 1);
            RemoveInternalChildRange(i, 1);
        }
    }
}
```
最后完成递归的MeasureCore过程。
```csharp
return new Size(double.IsInfinity(availableSize.Width) ? 0 : availableSize.Width, double.IsInfinity(availableSize.Height) ? 0 : availableSize.Height);
```
继续实现“布局”`ArrangeOverride(Size)`，把每一个`UIElementCollection`的子项放置在容器中的合适位置，这个同时还受布局方式的控制，我们可以选择把元素按左右对齐或居中的方式放置。
```csharp
public static readonly DependencyProperty HorizontalContentAlignmentProperty = DependencyProperty.RegisterAttached("HorizontalContentAlignment", typeof(WrapPanelAlignment), typeof(VirtualizingWrapPanel), new FrameworkPropertyMetadata(WrapPanelAlignment.Left, FrameworkPropertyMetadataOptions.AffectsMeasure | FrameworkPropertyMetadataOptions.AffectsArrange));

public WrapPanelAlignment HorizontalContentAlignment
{
    get => (WrapPanelAlignment)GetValue(HorizontalContentAlignmentProperty);
    set => SetValue(HorizontalContentAlignmentProperty, value);
}

protected override Size ArrangeOverride(Size finalSize)
{
    var generator = this.ItemContainerGenerator;

    UpdateScrollInfo(finalSize);

    for (int i = 0; i <= this.Children.Count - 1; i++)
    {
        var child = this.Children[i];
        int itemIndex = generator.IndexFromGeneratorPosition(new GeneratorPosition(i, 0));

        int childrenPerRow = CalculateChildrenPerRow(finalSize);
        int row = itemIndex / childrenPerRow;
        int column = itemIndex % childrenPerRow;

        double xCoordForItem = 0;
        // 不同的布局方式
        if (HorizontalContentAlignment == WrapPanelAlignment.Left)
            xCoordForItem = column * this.ChildWidth;
        else
        {
            if (childrenPerRow > this.Children.Count)
                childrenPerRow = this.Children.Count;
            double widthOfRow = childrenPerRow * this.ChildWidth;
            double startXForRow = finalSize.Width - widthOfRow;
            if (HorizontalContentAlignment == WrapPanelAlignment.Center)
                startXForRow /= 2;
            xCoordForItem = startXForRow + column * this.ChildWidth;
        }
        child.Arrange(new Rect(xCoordForItem, row * this.ChildHeight, this.ChildWidth, this.ChildHeight));
    }
    return finalSize;
}
```
剩下的就是一些扫尾工作，实现滚动条的位置随滚动变化。
```csharp
public void SetVerticalOffset(double offset)
{
    if (offset < 0 || this.viewport.Height >= this.extent.Height)
        offset = 0;
    else
        if (offset + this.viewport.Height >= this.extent.Height)
            offset = this.extent.Height - this.viewport.Height;

    this.offset.Y = offset;
    this.owner?.InvalidateScrollInfo();
    this.trans.Y = -offset;

    this.InvalidateMeasure();
}
```
重载基类的几个方法。
```csharp
protected override void OnRenderSizeChanged(SizeChangedInfo sizeInfo)
{
    base.OnRenderSizeChanged(sizeInfo);
    this.SetVerticalOffset(this.VerticalOffset);
}

protected override void OnClearChildren()
{
    base.OnClearChildren();
    this.SetVerticalOffset(0);
}

protected override void BringIndexIntoView(int index)
{
    if (index < 0 || index >= Children.Count)
        throw new ArgumentOutOfRangeException();

    int row = index / CalculateChildrenPerRow(RenderSize);
    SetVerticalOffset(row * ChildHeight);
}
```
最后补全`IScrollInfo`接口即可完成，此处未写出的方法全部`throw new InvalidOperationException()`。
```csharp
public void LineUp()
{
    this.SetVerticalOffset(this.VerticalOffset - this.ScrollOffset);
}

public void LineDown()
{
    this.SetVerticalOffset(this.VerticalOffset + this.ScrollOffset);
}

public void PageUp()
{
    this.SetVerticalOffset(this.VerticalOffset -
 this.viewport.Height);
}

public void PageDown()
{
    this.SetVerticalOffset(this.VerticalOffset + this.viewport.Height);
}

public void MouseWheelUp()
{
    this.SetVerticalOffset(this.VerticalOffset - this.ScrollOffset);
}

public void MouseWheelDown()
{
    this.SetVerticalOffset(this.VerticalOffset + this.ScrollOffset);
}

public Rect MakeVisible(Visual visual, Rect rectangle)
{
    return new Rect();
}
```
最终完成的代码样例可在Github上查看 [VirtualizingWrapPanel.cs](https://github.com/PeoLeser/WPFControls/blob/d42b73dd4caab73b1be28e7018f104ed3e9feb11/Digimezzo.WPFControls/VirtualizingWrapPanel.cs)。
不过这个容器仅仅实现了纵向的滚动，若想实现横向滚动，只需要修改Y轴为X轴即可。

##### <font color='grey'>Thanks for digimezzo</font> #####