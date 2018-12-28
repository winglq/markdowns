---
title: Oauth2 Authorization Code Grant模式分析 
date: 2018-12-28
draft: true
tags: ["oauth2", "golang"]
---

## OAuth2简介

OAuth2是一个将有限的HTTP服务访问权限提供给第三方应用的框架。OAuth2用于替换OAuth，OAuth2与OAuth不兼容，但是两者可以在一个系统中共存。

## OAuth2的角色

* **资源拥有者** 可以授予第三方服务访问受保护资源的权限。如果资源拥有者为个人，那么一般是指最终使用者。

* **资源服务器** 受保护资源所在的服务器。

* **客户端** 访问资源服务器的一个应用。只有在得到资源拥有者的授权后，客户端才能访问受保护的资源。

* **授权服务器** 访问令牌提供服务器。在资源拥有者授权客户端后，授权服务器会提供一个访问令牌给客户端，客户端后续访问资源服务器都要提供这个令牌。

比如有这样的场景，老王想要访问某论坛X,但是他又没有这个论坛的账号，而且他不想注册这个论坛的账号。这个论坛提供了通过Google账号访问的功能。于是老王就可以通过授权论坛X来获得他在谷歌的用户名并以此访问论坛X。
在这个场景中资源拥有者是老王，而他拥有的资源就是他在Google的账号。资源服务器就是Google的用户服务器。客户端就是论坛X的服务器。授权服务器就是Google的授权服务器。

## OAuth2 Authorization Code Grant模式流程

OAuth2提供了4种获取授权的模式，分别是Authorization Code Grant，Implicit Grant，Resource Owner Password Credentials Grant，Client Credentials Grant。这篇文章只分析第一种模式。
Code Grant是一个基于HTTP重定向的流程，客户端（应用程序）需要与用户的浏览器进行交互。从[RFC6749](https://tools.ietf.org/html/rfc6749)直接搬过来的流程图如下。

```
     +----------+
     | Resource |
     |   Owner  |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- & Redirection URI ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(D)-- Authorization Code ---------'      |
     |  Client |          & Redirection URI                  |
     |         |                                             |
     |         |<---(E)----- Access Token -------------------'
     +---------+       (w/ Optional Refresh Token)
```

各步骤的说明如下，为了方便理解上图中的User Agent使用浏览器代替。

A. 用户需要访问受保护资源的时候，客户端给浏览器发送一个重定向请求，重定向请求的query parameter包括一个认证成功后的回调地址，客户端的唯一标识符(client_id), 请求范围（request scope）, 本地状态（local state）。浏览器将页面跳转到授权服务器的登录页面。
B. 用户登录以确保该用户是资源拥有者。用户同意客户端所请求的权限(访问某个或某些资源)。
C. 授权服务器返回一个302请求给浏览器，请求的query parameter包括Authorization Code，State。302的location是在步骤A中设置的客户端的回调地址。浏览器跳转到location。
D. 客户端使用C中获取的Authorization Code，发送请求给授权服务器以获取访问令牌和更新令牌。请求参数中包括跟A中一样的回调地址，并提供客户端的client_id和client_secret。
E. 授权服务器接到请求后会认证客户端和步骤A中的客户端是同一个并且密码正确，检查Code，还要确保回调地址和C中的回调地址是同一个。如果所有检查都通过，授权服务器返回访问令牌和更新令牌（可选）。

我对这个流程比较困惑的是**为什么授权服务器不在步骤C直接在回调地址里包含两个令牌作为query parameter，而是要在E中才真正返回**。Google了下我的疑问，在Stack Overflow找到了[一个相同的问题](https://stackoverflow.com/questions/13387698/why-is-there-an-authorization-code-flow-in-oauth2-when-implicit-flow-works-s)，得分最高的回答解开了这个谜团。虽然OAuth2服务器必须使用HTTPS协议访问，但是浏览器和客户端之间无法保证是使用HTTPS的，因此如果在步骤C中返回令牌就很有可能被截获。因此在步骤C中只返回一个Authorization Code。即使这个Code和client_id被截获了，由于截获者没有client_secret，他也无法完成步骤E中的客户端认证，从而无法获取令牌。而任何一方跟授权服务器之间的访问由于使用了HTTPS而无法被截获。