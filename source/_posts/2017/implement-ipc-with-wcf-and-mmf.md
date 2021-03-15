---
title: 使用 WCF 和共享内存实现进程间通信
date: 2017-08-19 04:52:51
categories:
 - Tech
tags:
 - WCF
 - IPC
---

在Dopamine的issue中，有人提到了想让Dopamine暴露一组API以供Rainmeter调用。如果直接将`IPlaybackService`暴露出去是没有问题的，但是难以控制外部程序对其本身的修改，并且也无法广播通知提供播放状态的改变，相对于DDE和Socket，WCF不仅便于敏捷开发，而且其回调机制可以模拟事件。
<!--more-->

# 服务端部分 #
## 创建服务 ##
为了创建一个WCF服务，必须创建能够声明服务的接口和能够提供服务的、实现接口的实体类。

### 创建服务协定 ###
WCF要求在WCF中承载的服务接口必须具有`ServiceContractAttribute`特性，并且在接口中提供的方法必须具有`OperationContractAttribute`特性，于是对于一个服务接口而言，应该是这样的：
```csharp
[ServiceContract]
public interface IExternalControlServer
{
    [OperationContract]
    void Method();
}
```

如果要使服务能够支持回调，需要在`ServiceContractAttribute`中为回调协定指明回调接口的类型，并且回调接口需要和服务接口一样具有服务协定的特性。
```csharp
[ServiceContract(CallbackContract = typeof(IExternalControlServerCallback)]
public interface IExternalControlServer
{
    [OperationContract]
    void Method();
}

[ServiceContract]
public interface IExternalControlServerCallback
{
    [OperationContract]
    Task SendHeartBeatAsync();
}
```

### 创建服务行为 ###
对于实现服务接口的实体类而言，必须具有`ServiceBehaviorAttribute`特性以标记这个类是某个接口的对应实现，并且每一个实现的接口方法，也必须具有`OperationBehaviorAttribute`特性。
```csharp
[ServiceBehavior]
public class ExternalControlServer : IExternalControlServer
{
    [OperationBehavior]
    void Method(){}
}
```
而对于实现回调接口的实体类，其实现是在客户端完成而非服务端。

### 创建服务承载宿主 ###
WCF服务无法作为独立的程序单独运行，而是必须运行在能够承载WCF服务的宿主的上下文中，WCF支持多种承载方式，最常见的是使用IIS承载服务，也可以使用Windows服务主机承载，此处我希望该服务能够伴随Dopamine同时启动，故选择自承载。

`ServiceHost`类提供了一种简单的方式来承载WCF服务，在`address`中屏蔽了TCP、HTTP和管道等协议的差异，仅需要在创建宿主之后调用`Open()`方法即可启动服务。
由于是在本机上进行通信，因此选择功能略弱但更加便捷的命名管道。
```csharp
var host = new ServiceHost(typeof(ExternalControlServer), new Uri($"net.pipe://localhost/Dopamine"));
host.Open();
```
但是，由于我们需要实现回调，默认的方式无法满足我们的需求。在默认情况下，ServiceHost将会对每一个连接创建一个服务实例（这要求实体类必须有一个`public`的构造函数）和数据上下文。我们需要将服务设置为单例模式和可重入访问。
```csharp
[ServiceBehavior(InstanceContextMode = InstanceContextMode.Single, ConcurrencyMode = ConcurrencyMode.Reentrant)]
public class ExternalControlServer : IExternalControlServer
```
然后需要先实例化一个服务，然后用这个实例去实例化宿主`ServiceHost`，之后在宿主上设置终结点来绑定服务。
```csharp
var host = new ServiceHost(new ExternalControlServer(), new Uri($"net.pipe://localhost/Dopamine"));
hHost.AddServiceEndpoint(typeof(IExternalControlServer), new NetNamedPipeBinding(), "/ExternalControlService");
```

## 创建共享内存 ##
为了把FFT的数据传递给客户端，虽然可以直接通过WCF传递一个byte[]数组来解决，但是这会造成多次内存复制，不够高效也不够优雅，若使用共享内存则可以即高效又优雅。但是需要自己来解决访问的竞争。

首先创建一个互斥量。
```csharp
var mmfMutex = new Mutex(true, "DopamineFftDataMemoryMutex");
```
创建一个ACL规则，只给予客户端读权限，阻止客户端写入共享内存。
```csharp
var sec = new MemoryMappedFileSecurity();
sec.AddAccessRule(new AccessRule<MemoryMappedFileRights>(new SecurityIdentifier(WellKnownSidType.SelfSid, null), MemoryMappedFileRights.FullControl, AccessControlType.Allow));
sec.AddAccessRule(new AccessRule<MemoryMappedFileRights>(new SecurityIdentifier(WellKnownSidType.WorldSid, null), MemoryMappedFileRights.Read, AccessControlType.Allow));
```
使用该规则和指定的大小来分配内存。
```csharp
// FftDataLength是FFT数据占用的内存大小
var mmf = MemoryMappedFile.CreateNew("DopamineFftDataMemory", FftDataLength, MemoryMappedFileAccess.ReadWrite, MemoryMappedFileOptions.DelayAllocatePages, sec, HandleInheritability.None);
// 需要将共享内存打开为一个流
var mmfs = fftDataMemoryMappedFile.CreateViewStream(0, FftDataLength, MemoryMappedFileAccess.ReadWrite);
// 用二进制的数据写入流
var mmfsw = new BinaryWriter(fftDataMemoryMappedFileStream);
```
每次当FFT数据更新时，即可先加锁，然后直接写入共享内存中，再释放锁，客户端无需与服务端通信直接从内存中读取即可。
```csharp
mmfMutex.WaitOne();
mmfsw.Seek(0, SeekOrigin.Begin);
mmfsw.Write(fftDataBufferBytes);
mmfMutex.ReleaseMutex();
```

## 在一个宿主上承载多个服务 ##
为了服务的模块化，我将控制服务和FFT服务分成了两个接口，但是根据前文所写，`ServiceHost`中终结点的接口是与服务实例对应的，那如何去用多个接口添加多个终结点呢？如果不去创建多个宿主，那么只能让服务实体类去实现多个接口，如果将实体类用部分定义类去写，也并不会使得这个类过于庞大。
```csharp
[ServiceBehavior(InstanceContextMode = InstanceContextMode.Single, ConcurrencyMode = ConcurrencyMode.Reentrant)]
public class ExternalControlServer : IExternalControlServer, IFftDataServer
{
    // some codes
}

var host = new ServiceHost(svcExternalControlInstance, new Uri($"net.pipe://localhost/Dopamine"));
host.AddServiceEndpoint(typeof(IExternalControlServer), new NetNamedPipeBinding(), "/ExternalControlService");
host.AddServiceEndpoint(typeof(IFftDataServer), new NetNamedPipeBinding(), "/ExternalControlService/FftDataServer");
```

## 使用代码发布服务的元数据 ##
在服务启动之后，如果使用WCF Test Client，会发现无法取得服务的任何信息，无法在解决方案管理器中添加服务引用，而不得不去复制接口，非常不便。因此，需要让服务能够发布元数据。

发布元数据依赖的是名为“Mex绑定”的一个东西，所谓Mex指的是WS-MetadataExchange (Web 服务元数据交换)，通过其GetMetadata请求来交换由WS-Policy信息注释的Web Service描述语言 (WSDL)。听起来很复杂，不过复杂的过程WCF已经封装起来了，只需要添加一个mex终结点即可完成一切。
```csharp
var smb = host.Description.Behaviors.Find<ServiceMetadataBehavior>() ?? new ServiceMetadataBehavior();
smb.MetadataExporter.PolicyVersion = PolicyVersion.Policy15;
host.Description.Behaviors.Add(smb);
host.AddServiceEndpoint(ServiceMetadataBehavior.MexContractName, MetadataExchangeBindings.CreateMexNamedPipeBinding(), "/ExternalControlService/mex");
```

## 向客户端广播通知 ##
为了能够调用客户端的回调方法，必须要保存一个对客户端的引用，因此我在服务端公开了一个`RegisterClient()`方法用来让客户端在服务端上注册。
```csharp
[OperationBehavior(ReleaseInstanceMode = ReleaseInstanceMode.None)]
public string RegisterClient()
{
    var context = OperationContext.Current;
    var sessionId = context.SessionId;
    try
    {
        var callback = context.GetCallbackChannel<IExternalControlServerCallback>();
        // clients 是一个哈希表，用来保存所有注册的客户端
        clients.TryRemove(context.SessionId);
        clients.Add(sessionId, callback);
        return sessionId;
    }
    catch (Exception)
    {
        clients.TryRemove(sessionId);
        return string.Empty;
    }
}
```
当我想给客户端发送通知时（也就是调用回调方法），和调用实例方法是一样的，直接在服务端执行回调接口中的方法就会调用对应实例的方法，但是此时已经跨进程了，是不是很方便？
```csharp
foreach (var client in clients)
    client.CallbackMethod();
```

# 客户端部分 #
## 添加服务引用 ##
在解决方案管理器中选择一个项目，选择添加服务引用，在Address中填上mex终结点的地址，也就是“net.pipe://localhost/Dopamine/ExternalControlService/mex”，即可自动获取所有元数据（包括自定义类型）。
![add-service-reference](/content/images/2017/08/add-service-reference.png)

## 与服务端连接 ##
这样就直接可以引用相关的API与服务端交互，不过要先建立一个WCF连接：
```csharp
// 创建实体类
public class ExternalControlClientCallback : IExternalControlServerCallback
{
    public void SendHeartBeat(){}
}

public static class ExternalControlServerFactory
{
    var factory = new DuplexChannelFactory<IExternalControlServer>(new InstanceContext(new ExternalControlClientCallback()), new NetNamedPipeBinding(), new EndpointAddress("net.pipe://localhost/Dopamine/ExternalControlService"));
    var server = factory.CreateChannel();
    
    server.RegisterClient();
    //server.DoSomething();
}
```

## 读取共享内存 ##
在服务端中创建的共享内存和互斥量实际上是一个内核对象，内核对象对任何进程都是可见的而并不是在创建者进程外就无法访问的，只要打开的对象名和服务端相同，就可以访问到同样的对象。
```csharp
// 由于在服务端限制了可访问性，因此客户端必须指定 Read 权限
var fftData = new byte[FftDataLength];
var mmf = MemoryMappedFile.OpenExisting("DopamineFftDataMemory", MemoryMappedFileRights.Read);
var mmfs = map.CreateViewStream(0, FftDataLength, MemoryMappedFileAccess.Read);
var mmfsr = new BinaryReader(mapViewStream);
mmfs.Seek(0, SeekOrigin.Begin);
fftData = mmfsr.ReadBytes(fftData.Length);
```

相关的源码：
[服务端](https://github.com/Magentaize/Dopamine/tree/master/Dopamine.Common/Services/ExternalControl)
[客户端](https://github.com/Magentaize/Dopamine-ExternalControl-Support)