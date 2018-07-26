---
title: C++ WebSocket 库
date: 2018-07-1625 20:43:49
categories: 网络技术
tags:
- 网络技术
---

WebSocket 是 HTML5 的一个引入注目的特性，它通常用于 Web 端，为构建实时的 Web 应用提供方便。WebSocket 是一个基于 TCP 的协议，它借助于 HTTP 请求，建立客户端与服务器端之间的双向通道，通道建立完成后，客户端和服务器端都可以通过这条通道方便地收发消息，因而 WebSocket 一向有着 “Web 的 TCP” 之称。

WebSocket 不是 JavaScript 的一个接口，而是一个定义良好的基于消息的协议。得益于不同平台对于 WebSocket 协议的广泛实现，它更为跨多种平台的 ***实时网络应用程序*** 开发提供了极大的方便。除了可以在前端开发的 JavaScript 中使用 WebSocket 之外，我们也可以在 Java、C++、Go、Rust 等编程语言平台中使用 WebSocket。
<!--more-->
`uWebSockets` 是一个 C/C++ 的 WebSocket 库，它的 [GitHub 主页](https://github.com/uNetworking/uWebSockets) 列出了一些常见的 WebSocket 实现库的对比，如下图：
![overview.png](https://www.wolfcstech.com/images/1315506-4499065962276452.png)

其中，ws-rs，[项目主页](https://ws-rs.org/)，[GitHub 主页](https://github.com/housleyjk/ws-rs)，是一个轻量级的，事件驱动的用于 Rust 的 WebSocket 库。Gorilla，[项目主页](http://www.gorillatoolkit.org/)，[GitHub 主页](https://github.com/gorilla)，是 Go 语言的 Web 工具包，它包含了 WebSocket 的实现，WebSocket 实现的 [GitHub 主页](https://github.com/gorilla/websocket)。websockets，[项目主页](https://websockets.readthedocs.io/en/stable/)，[GitHub 主页](https://github.com/aaugustin/websockets/)，是一个 Python 的 WebSocket 实现。Socket.IO，[项目主页](https://socket.io/)，[GitHub 主页](https://github.com/socketio)，主要是 Node.JS 服务器的实时应用框架，其中包含了 WebSocket 的实现。其它库则都是 C/C++ 的 WebSocket 实现。

从中我们可以捞到 uWebSockets、Crow、websocketpp、Beast 这样几个 C/C++ 的 WebSocket 库。此外，还有 libwebsockets 和 POCO 库的 WebSocket 模块可以用。这里汇总已知的可以在 C++ 中使用的 WebSocket 库。

## uWebSockets

[GitHub 主页](https://github.com/uNetworking/uWebSockets)

uWebSockets，µWS ("microWS") 是一个客户端和服务器的 WebSocket 和 HTTP 实现。它简单、高效且轻量级。

## Crow

[GitHub 主页](https://github.com/ipkn/crow)

Crow 是一个 Web 微框架。

## websocketpp（WebSocket++）

[GitHub 主页](https://github.com/zaphoyd/websocketpp)
[项目主页](https://www.zaphoyd.com/websocketpp)

websocketpp 是 C++ 的 WebSocket 客户端/服务器库。它是一个开源的只包含头文件的 C++ 库，它实现了 RFC6455 WebSocket 协议。它允许向 C++ 程序中集成 WebSocket 客户端和服务器功能。它使用可交换的网络传输模块，包括基于 C++ iostreams 的和基于 [Boost Asio](http://www.boost.org/doc/libs/1_48_0/doc/html/boost_asio.html) 的。

## Beast

[GitHub 主页](https://github.com/boostorg/beast)
[项目主页](http://www.boost.org/doc/libs/1_66_0/libs/beast/doc/html/index.html)

基于 Boost.Asio 以 C++11 构建的 HTTP 和 WebSocket 库。Boost 项目的 HTTP 和 WebSocket 库。

## Poco Websocket

[GitHub 主页](https://github.com/pocoproject/poco)
[项目主页](https://pocoproject.org/)

POCO C++ 库是一个跨平台的 C++ 网络库。其中包含了 WebSocket 的实现模块。

## libwebsockets

[GitHub 主页](https://github.com/warmcat/libwebsockets)

规范 libwebsockets.org websocket 库

参考资料：

[C++ WebSocket++ 的Client使用详解](http://www.myhack58.com/Article/68/2014/51982.htm)

[基于C/C++的WebSocket库](https://blog.gmem.cc/websocket-library-for-c-or-cpp)

Done.
