---
title: 调用CreateViewStream()时access参数不能为Write的坑
date: 2017-08-14 02:00:41
categories: Tech
tags:
- NetFx
- IPC
---

当为IPC调用使用`MemoryMappedFile.CreateNew()`方法创建共享内存时，其可访问性级别至少要设置为`MemoryMappedFileAccess.ReadWrite`，但是，当调用`MemoryMappedFile.CreateViewStream(long offset, long size, MemoryMappedFileAccess access)`为共享内存创建流的时候，却不能将`access`的可访问性设置为`MemoryMappedFileAccess.Write`，而必须设置为`MemoryMappedFileAccess.ReadWrite`，这一点在 MSDN 中并没有明确指明，但是`MemoryMappedFileAccess`的注释中却说当创建 View 的时候`Write`级别是可以被设置的：
<!--more-->
```csharp
// This enum maps to both the PAGE_XXX and FILE_MAP_XXX native macro definitions.
// It is used in places that check the page access of the memory mapped file. ACL
// access is controlled by MemoryMappedFileRights.
[Serializable]
public enum MemoryMappedFileAccess {
    ReadWrite = 0,
    Read,
    Write,   // Write is valid only when creating views and not when creating MemoryMappedFiles   
    CopyOnWrite,
    ReadExecute,
    ReadWriteExecute,
}
```
在`CreateViewStream()`方法中检查过可访问性之后，会调用`CreateView()`来尝试获得句柄：
```csharp
[System.Security.SecurityCritical]
[SecurityPermissionAttribute(SecurityAction.Demand, Flags = SecurityPermissionFlag.UnmanagedCode)]
public MemoryMappedViewStream CreateViewStream(Int64 offset, Int64 size, MemoryMappedFileAccess access) {
 
    if (offset < 0) {
        throw new ArgumentOutOfRangeException("offset", SR.GetString(SR.ArgumentOutOfRange_NeedNonNegNum));
    }
 
    if (size < 0) {
        throw new ArgumentOutOfRangeException("size", SR.GetString(SR.ArgumentOutOfRange_PositiveOrDefaultSizeRequired));
    }
 
    if (access < MemoryMappedFileAccess.ReadWrite || access > MemoryMappedFileAccess.ReadWriteExecute) {
        throw new ArgumentOutOfRangeException("access");
    }
 
    if (IntPtr.Size == 4 && size > UInt32.MaxValue) {
        throw new ArgumentOutOfRangeException("size", SR.GetString(SR.ArgumentOutOfRange_CapacityLargerThanLogicalAddressSpaceNotAllowed));
    }
 
    MemoryMappedView view = MemoryMappedView.CreateView(_handle, access, offset, size);
    return new MemoryMappedViewStream(view);
}
```
随后，在`CreateView()`中巨硬帮我们进行了内存页对齐，32位长整数指针判断，请求空间可用判断巴拉巴拉的，由于我们申请的内存大小可能小于物理内存大小，部分内容会被放在分页文件里，为了保证 View 的可用性和一致性，用到了`VirtualAlloc()`将参数设置为`MEM_COMMIT`来返回“提交”的内存指针，并且使用`GetPageAccess()`来将`MemoryMappedFileAccess`转换为`flProtect`。

但是，在[内存可访问性常量](https://msdn.microsoft.com/en-us/library/windows/desktop/aa366786(v=vs.85).aspx)中，并没有能与“只写(write-only)”相对应的可访问性级别，也就是说，注释里的：
>This enum maps to both the PAGE_XXX and FILE_MAP_XXX native macro definitions.

并不是和实际情况一样, `Write`只能被用在共享内存文件视图中。