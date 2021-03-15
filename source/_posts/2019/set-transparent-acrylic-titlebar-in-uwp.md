---
title: Set Transparent Acrylic TitleBar in UWP
date: 2019-03-01 02:48:13
categories:
 - Tech
tags:
 - UWP
---

Nowadays, more and more UWP applications have used Fluent Design System such as acrylic theme brush, that official's Groove Music, Calculator and 3-rd's ithome, Aurora Music, although they all have a transparent title bar, their workarounds aren't the same. Due to that I'm creating a UWP version of Dopamine, finding out how to make title bar acrylic is my first task, so I dig into it.
<!--more-->

If you have Google it, MSDN will tell you that just to set ExtendViewIntoTitleBar to true. But hold on please, what's the next? To set frame's background to acrylic? You'll have no idea about what to do next and per-app has its self logical. I've compared those apps one by one, they have not only different height of title bar but also non-unified sizes of navigation button and three-point menu button. The reason of that they don't behave the same is Microsoft doesn't provide an official workaround.

Not as same as WPF, an UWP app not only has the window but also has a navigation service. Each time you jump from a page to another, the previous page will be pushed into "navigation stack", if the navigation service is enabled and the button of it is visible, once you press it, current page will be hidden and the previous one will be shown. Actually, the whole logical of it is very complex so I selected Prism as the base framework. But it brings a new problem that Prism will create a frame as root frame itself in its private method, that make it's hard to construct a ribbon layout.

```xml
<Page
  Background="{ThemeResource SystemControlAcrylicWindowBrush}">

  <StackPanel>
    <Border Height="28" >
      <TextBlock x:Name="TitleBar" 
                 Text="Fluent Player" 
                 FontSize="12" 
                 VerticalAlignment="Center" 
                 Margin="10,0,0,0"/>
    </Border>
    <Frame x:Name="ShellFrame"/>
  </StackPanel>
</Page>
```

```csharp
private void ExtendView()
{
    CoreApplication.GetCurrentView().TitleBar.ExtendViewIntoTitleBar = true;
    var titleBar = ApplicationView.GetForCurrentView().TitleBar;
    titleBar.ButtonBackgroundColor = Colors.Transparent;
    titleBar.ButtonInactiveBackgroundColor = Colors.Transparent;
}

private void ReplaceNavigationService()
{
    var _shell = new Shell();
    var frame = _shell.FindName("ShellFrame") as Frame;
    var logger = Container.Resolve<ILoggerFacade>();
    var frameFacade = new FrameFacade(frame, logger);
    Container.GetContainer().UseInstance(typeof(IFrameFacade), frameFacade, IfAlreadyRegistered.Replace);
    var nav = new NavigationService(logger, frameFacade);
    Container.GetContainer().UseInstance(typeof(IPlatformNavigationService), nav, IfAlreadyRegistered.Replace);
    NavigationService = nav;
    _shell.DataContext = new ShellViewModel();
    Window.Current.Content = _shell;
}
```

Fortunately, in the new version of Prism, its team has used DI to build navigation service and our mission only left with replacing this service. There's a `IFrameFacade` inside original `IPlatformNavigationService` and there's a `Frame` inside original `IFrameFacade` so I first use origin service to set my ShellPage, next extract the child frame, finally create their instance and inject them into the IOC container. Through these operations the Prism's original navigation frame have been replaced.
