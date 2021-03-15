---
title: 解决无边框WindowChrome无法触发自动隐藏的任务栏
date: 2017-06-03 07:21:20
categories:
 - Tech
tags: WPF
---

在 WPF 中，为了自己实现无边框窗口，需要使用 `WindowChrome` 来实现，但是，这会导致当窗口全屏时，如果任务栏是自动隐藏的，则当窗口被激活时，无法触发显示任务栏。为了解决这个问题，可以把距离任务栏最近的一个方向减少1px，这样一来任务栏就不会被遮挡。
但是这个方法并不是足够完美，Office 全家桶和 Visual Studio 就没有使用这种方式。
<!--more-->

我们会选择在`InitializeWindow()`方法中来实例化WindowChrome，如同以下代码，注意添加了窗口大小改变的handler
```csharp
protected void InitializeWindow()
{
    this.windowChrome = new WindowChrome
    {
        CaptionHeight = 0,
        CornerRadius = new CornerRadius(0)
    };

    WindowChrome.SetWindowChrome(this, this.windowChrome);

    this.SizeChanged += this.BorderlessWindow_SizeChanged;
}
```
当窗口大小改变时，在handler中判断`WindowState`，如果是`WindowState.Maximized`就需要对窗体大小进行改变
```csharp
protected virtual void BorderlessWindowBase_SizeChanged(object sender, SizeChangedEventArgs e)
{
    if (this.WindowState == WindowState.Maximized)
        FixWindowSize();
}
```
WPF没有封装修改窗口大小的方法，这需要用到几个Native Method
```csharp
[SuppressUnmanagedCodeSecurity]
internal static partial class NativeMethods
{
    [DllImport("user32.dll")]
    [return: MarshalAs(UnmanagedType.Bool)]
    internal static extern bool SetWindowPos(IntPtr hWnd, IntPtr hWndInsertAfter, int X, int Y, int cx, int cy, uint uFlags);

    [DllImport("user32.dll", EntryPoint = "GetMonitorInfoW",         ExactSpelling = true, CharSet = CharSet.Unicode)]
    [return: MarshalAs(UnmanagedType.Bool)]
    internal static extern bool GetMonitorInfo(IntPtr hMonitor, [Out] MONITORINFO lpmi);

    [DllImport("user32.dll")]
    internal static extern IntPtr MonitorFromWindow(IntPtr handle, uint flags);

    [DllImport("shell32.dll")]
    internal static extern uint SHAppBarMessage(int dwMessage, ref APPBARDATA pData);

    [DllImport("user32.dll", EntryPoint = "SetWindowLongPtr")]
    [SuppressMessage("Microsoft.Interoperability", "CA1400:PInvokeEntryPointsShouldExist", Justification = "Entry point does exist on 64-bit Windows.")]
    internal static extern IntPtr SetWindowLongPtr64(IntPtr hWnd, Int32 nIndex, IntPtr dwNewLong);

    [DllImport("user32.dll")]
    internal static extern Int32 GetWindowLong(IntPtr hWnd, int nIndex);
}
```
需要用到的数据类型
```csharp
[StructLayout(LayoutKind.Sequential)]
internal struct APPBARDATA
{
    public int cbSize;
    public IntPtr hWnd;
    public int uCallbackMessage;
    public int uEdge;
    public RECT rc;
    public bool lParam;
}

[StructLayout(LayoutKind.Sequential)]
internal class MONITORINFO
{
    public int cbSize = Marshal.SizeOf(typeof(MONITORINFO));
    public RECT rcMonitor = new RECT();
    public RECT rcWork = new RECT();
    public int dwFlags = 0;
}
```
需要用到的常量
```csharp
internal const uint MONITOR_DEFAULTTONEAREST = 0x2;
internal const int HWND_NOTOPMOST = -2;
internal const uint SWP_SHOWWINDOW = 0x40;

internal enum ABMsg
{
    ABM_GETSTATE = 4,
    ABM_GETTASKBARPOS = 5
}

internal enum ABEdge
{
    ABE_LEFT = 0,
    ABE_TOP = 1,
    ABE_RIGHT = 2,
    ABE_BOTTOM = 3
}

internal enum GWL
{
    STYLE = -16
}

internal enum WS
{
    SYSMENU = 0x80000
}
```
执行的`FinWindowSize()`函数
```csharp
internal void FixWindowSize()
{
    var mHwnd = new WindowInteropHelper(this).Handle;
    var monitor = NativeMethods.MonitorFromWindow(mHwnd, MONITOR_DEFAULTTONEAREST);

    var pData = new APPBARDATA();
    pData.cbSize = Marshal.SizeOf(pData);
    pData.hWnd = mHwnd;

    // 判断任务栏是否自动隐藏
    if (Convert.ToBoolean(NativeMethods.SHAppBarMessage((int)ABMsg.ABM_GETSTATE, ref pData)))
    {
        if (monitor != IntPtr.Zero)
        {
            // 获取当前屏幕大小
            var monitorInfo = new MONITORINFO();
            NativeMethods.GetMonitorInfo(monitor, monitorInfo);
            int x = monitorInfo.rcWork.left;
            int y = monitorInfo.rcWork.top;
            int cx = monitorInfo.rcWork.right - x;
            int cy = monitorInfo.rcWork.bottom - y;

            // 获取任务栏位置
            NativeMethods.SHAppBarMessage((int)ABMsg.ABM_GETTASKBARPOS, ref             pData);
            var uEdge = GetEdge(pData.rc);

            switch (uEdge)
            {
                case ABEdge.ABE_TOP: y++;break;
                case ABEdge.ABE_BOTTOM: cy--;break;
                case ABEdge.ABE_LEFT: x++;break;
                case ABEdge.ABE_RIGHT: cx--;break;
            }
            // 设置窗体大小
            NativeMethods.SetWindowPos(mHwnd, new IntPtr(Constants.HWND_NOTOPMOST), x, y, cx, cy,SWP_SHOWWINDOW);

            // 去除Windows Form标题栏
            // 虽然一般情况下WindowChrome不会显示出标题栏以及右上角三个标准按钮，但是在调用SetWindowPos之后标题栏又会重新出现
            NativeMethods.SetWindowLongPtr(hwnd, Convert.ToInt32(GWL.STYLE), (IntPtr)(NativeMethods.GetWindowLong(hwnd, Convert.ToInt32(GWL.STYLE)) & ~Convert.ToInt32(WS.SYSMENU)));
        }
    }
}

private static ABEdge GetEdge(RECT rc)
{
    ABEdge uEdge;
    if (rc.top == rc.left && rc.bottom > rc.right)
        uEdge = ABEdge.ABE_LEFT;
    else if (rc.top == rc.left && rc.bottom < rc.right)
        uEdge = ABEdge.ABE_TOP;
    else if (rc.top > rc.left)
        uEdge = ABEdge.ABE_BOTTOM;
    else
        uEdge = ABEdge.ABE_RIGHT;
    return uEdge;
}
```