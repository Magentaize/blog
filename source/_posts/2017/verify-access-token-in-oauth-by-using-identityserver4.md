---
title: 使用IdentityServer4完成OAuth2.0的access_token验证
date: 2017-11-05 11:53:49
categories: Tech
tags:
 - ASP.NET
---

在ASP.NET Core中，用于实现OAuth认证与单点登录的框架并不是很多，大体上有IdentityServer和OpenIddict以及官方的Microsoft.AspNetCore.Authorization。在某些情况下，我们并不想去做一个OAuth的客户端，而是要做一个服务端的实现，而这就比较复杂了，本文将覆盖对Bearer Token验证的几个坑。
<!--more-->

### 前置内容
很多框架的文档以及博客中，都只是对OAuth2.0这个协议有一个抽象概念的解析，但是在OAuth2.0的具体实现中，并不是如同很多流程图中所描述的“Client向RS请求资源时，RS会验证Client发来的经RO授权的由AuthZ颁发的access_token”一般简单。这个验证过程，抛去如何获得access_token的过程不谈，就算把access_token连同请求一起提交给RS就有至少三种途径。当然，最终选择的方式肯定是由RS确定，但是在验证token这个过程中，有很多非常琐碎的细节值得注意。

这里又会牵扯一个新的概念，叫做JWT (JSON Web Token)，在[RFC 7519](https://tools.ietf.org/html/rfc7519)中提出。一个JWT由**头部**、**载荷**与**签名**组成，我们无需关心JWT的实际组织结构，只需要知道JWT代表了一个用户和服务器间**已经建立的认证关系**即可，并且JWT最后附上了一段签名来确保JWT的可靠性。

那么，JWT显然是一段JSON，这个JSON是如何传递给RS的呢？是用URL Encoder吗？
显然不是，但是OAuth2.0协议([RFC 6749](http://tools.ietf.org/html/rfc6749))中并没有对如何传递Token做出规定，因此还有另一个协定([RFC 6750](http://tools.ietf.org/html/rfc6750))来指导如何传递JWT。在RFC 6750中，用Bearer Token这一术语来描述传递与验证Token的方法。但是需要注意的是，RFC 6750并不属于OAuth2.0协议，而仅是一个指导性协定，最终的产品可以选择使用Bearer来作为具体实现，也可以确立一套自成体系的协议。

### 添加验证
在RFC 6750中，推荐使用HTTP Header与Request Body方式来传递Bearer，对于客户端的实现在此不便累述，以下均假设使用HTTP Header方式，也就是说Client的请求头中包含有
```html
GET /resource HTTP/1.1
Host: server.example.com
Authorization: Bearer <token>
```
对于IdentityServer4的默认配置，只有默认的`IdentityServer4.Validation.JwtBearerClientAssertionSecretParser`来对Request Body进行Bearer Token的验证。如果想要加入HTTP Header方式的验证，需要安装一个包叫`IdentityServer4.AccessTokenValidation`，这个包在内部配置了`Microsoft.AspNetCore.Authentication`并使用`Microsoft.AspNetCore.Authentication.JwtBearer`来对通过HTTP Header传递进来的Bearer进行验证，这一切只需要添加一行即可：
```csharp
services.AddAuthentication(IdentityServerAuthenticationDefaults.AuthenticationScheme).AddIdentityServerAuthentication();
```
是不是感觉非常易用？梦里啥都有。
然而真正运行之后，Microsoft.AspNetCore.Authentication将会抛出一个为`IDX10500: Signature validation failed. No security keys were provided to validate the signature.`的异常，因为没有配置证书，无法对JWT的签名进行验证。

什么？我不是能生成JWT吗，怎么会没有证书去验证呢？
在很多情况下，AuthN和AuthZ并不是同一台服务器，生成JWT只是说明AuthN拥有了证书，而AuthZ需要去验证AuthN的签名，并且无法与AuthN直接通信。在我们这种全部服务器都跑在一个应用中的场景，仍然需要与分布式一样配置，也就是为AuthZ添加一个证书。

我希望的是IdentityServer能为包IdentityServer4.AccessTokenValidation内置添加证书的功能，这也更加符合直觉，既然IdentityServer4.AccessTokenValidation已经能验证JWT了，为什么我还要去使用更加原生`Microsoft.AspNetCore.Authorization来单独添加一个证书呢？

在IdentityServer4.AccessTokenValidation的源码中的IdentityServerAuthenticationOptions.cs文件的`void ConfigureJwtBearer(JwtBearerOptions jwtOptions)`函数中，我们来硬编码添加一个证书：
```csharp
internal void ConfigureJwtBearer(JwtBearerOptions jwtOptions)
{
    /* some codes */

    var filename = Path.Combine(Directory.GetCurrentDirectory(), "tempkey.rsa");
    var keyFile = File.ReadAllText(filename);
    var tempKey = JsonConvert.DeserializeObject<TemporaryRsaKey>(keyFile);
    var rsa = new RsaSecurityKey(tempKey.Parameters) {KeyId = tempKey.KeyId};
    jwtOptions.TokenValidationParameters.IssuerSigningKey = rsa;
    jwtOptions.TokenValidationParameters.ValidIssuer = "http://localhost:5000";
        
    /* some codes */
}
```
这里硬编码的tempkey.rsa是IdentityServer4默认的开发环境证书，也就是使用`services.AddDeveloperSigningCredential()`创建的证书，而下面读取证书的方法也和这个扩展方法中的内容一样。如果需要设置一个开发环境证书的话，记得为`IdentityServerAuthenticationOptions`新增一个属性来传递证书。
`ValidIssuer`的值是生成JWT的服务器地址，既然我们是单应用程序，那就和应用域名一样了。

再次运行，可以看到结果
```html
[22:12:40 Information] Microsoft.AspNetCore.Authentication.JwtBearer.JwtBearerHandler
Successfully validated the token.

[22:12:40 Information] Microsoft.AspNetCore.Authentication.JwtBearer.JwtBearerHandler
AuthenticationScheme: BearerIdentityServerAuthenticationJwt was successfully authenticated.

[22:12:40 Information] IdentityServer4.AccessTokenValidation.IdentityServerAuthenticationHandler
AuthenticationScheme: Bearer was successfully authenticated.
```
说明access_token已成功验证。