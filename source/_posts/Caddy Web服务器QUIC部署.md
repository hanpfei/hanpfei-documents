---
title: Caddy Web服务器QUIC部署
date: 2017-01-09 16:05:49
tags:
- 服务器
- 网络
---

# Caddy 简介
Caddy是一个Go语言写的，易于使用的通用Web服务器。它具有如下的一些功能：
* **配置简单**：Caddy服务器的运行可以通过Caddyfile配置文件进行配置，Web服务配置起来非常简单。
* **自动的HTTPS**：它可以自动地为我们申请 [Let's Encrypt](https://letsencrypt.org/) 域名证书，管理所有的密码学设施，并进行配置。
* **HTTP/2**：默认支持HTTP/2（由Go标准库支持）
<!--more-->
* **虚拟主机托管**：Caddy支持TLS的SNI。SNI是在2006年加入TLS的一个TLS扩展。客户端在TLS握手的Client Hello消息中，通过SNI扩展将请求的资源的域名发送给服务器，服务器根据SNI的域名来下发TLS证书。这样就可以在具有单个公网IP的同一台主机上部署多个不同的域名的服务。可以为Caddy服务的不同域名配置不同的证书和密钥。
* **QUIC支持**：Caddy实验性地支持QUIC协议，以获取更好的性能。
* TLS session ticket **key rotation** for more secure connections
* **良好的可扩展性**：因此Caddy非常方便针对自己的需求做定制。
* **随处运行**：这主要与Go应用程序的特性有关。Go的模块都被编译为静态库，这使得Go的应用程序在编译为可执行文件时都是静态链接的，因而依赖的动态库极少，这使得部署使用非常方便。

自动的HTTPS、HTTP/2支持、QUIC支持和随处运行这些特性非常有吸引力，特别是对QUIC的支持。

此外，Caddy的性能非常好。下面两幅图是我的静态个人博客站点，分别是用Caddy和nginx作为Web服务器，打开主页所需的加载时间对比：
![Service with Caddy](https://www.wolfcstech.com/images/1315506-9da3094340e8363f.png)

![Service with nginx](https://www.wolfcstech.com/images/1315506-b326960e37658396.png)

上面的图显示了以Caddy作为Web服务器，主页的加载时间只有680ms；下面的图显示以nginx作为Web服务器，主页的加载时间则长达1.99s，要慢接近2倍。

# Caddy部署
Caddy应用程序不依赖于其它组件，且官方已经为不同的平台提供了二进制可执行程序。可以通过如下三种方式之一安装Caddy：
* 在 [下载页](https://caddyserver.com/download)，通过浏览器定制自己需要的功能集，并下载相应的二进制可执行程序。
* 预编译的 [最新发行版](https://github.com/mholt/caddy/releases/latest) 二进制可执行程序。
* curl [getcaddy.com](https://getcaddy.com/) 来自动安装：`curl https://getcaddy.com | bash`。

将caddy的路径加如PATH环境变量中。之后可以 `cd` 进入网站的文件夹，并运行 `caddy`来提供服务。默认情况下，Caddy在2015端口上为网站提供服务。

要定制网站提供服务的方式，可以为网站创建名为**Caddyfile**的文件。当运行 `caddy` 命令时，它会自动地在当前目录下寻找并使用Caddyfile文件来为自己做配置。

要了解更多关于Caddyfile文件的写法，可以参考 [Caddyfile 文档](https://caddyserver.com/docs/caddyfile)。

注意生产环境网站默认是通过HTTPS提供服务的。

Caddy还有命令行接口。运行`caddy -h` 可以查看基本的帮助信息，或参考 [CLI文档](https://caddyserver.com/docs/cli) 来了解更多详情。

**以Root运行**：建议不要这样做。但依然可以通过像这样使用setcap来监听端口号小于1024的端口：`sudo setcap cap_net_bind_service=+ep ./caddy`

# 由源码运行
注意：需要安装 **[Go 1.7](https://golang.org/dl/)**或更新的版本才可以。
1. `go get github.com/mholt/caddy/caddy`
2. `cd` 进入网站的目录
3. 执行`caddy`（假设 `$GOPATH/bin` 已经在 `$PATH` 中了）

Caddy的 `main()` 再caddy子目录下。要编译Caddy，可以使用在那个目录下找到的 `build.bash`。

# 在生产环境运行
Caddy项目官方不维护任何系统特有的集成方法，但下载的文档中包含了社区共享的 [非官方资源](https://github.com/mholt/caddy/tree/master/dist/init)，用以帮助在生产环境运行Caddy。

以何种方式运行Caddy全由自己决定。许多用户使用 `nohup caddy &` 就可以满足需求了。其他人使用 `screen`。有些用户需要再重启之后就运行Caddy，可以在触发重启的脚本中来做到这一点，通过给init脚本添加一个命令，或给操作系统配置一个service。

可以看一下我的个人博客站点的完整Caddyfile内容：
```
wolfcstech.com:80 www.wolfcstech.com:80 {
    root /home/www-data/www/hanpfei-documents/public
    redir 301 {
        if {>X-Forwarded-Proto} is http
        /  https://{host}{uri}
    }
}

wolfcstech.com:443 www.wolfcstech.com:443 {
        tls /home/www-data/www/ssl/chained.pem /home/www-data/www/ssl/domain.key
        #tls test@admpub.com
        root /home/www-data/www/hanpfei-documents/public
        gzip
        log ../access.log
}
```

# 启用QUIC
Caddy 0.9 已经实验性地提供了对QUIC的支持，这主要通过 [lucas-clemente/quic-go](https://github.com/lucas-clemente/quic-go) 来实现。要尝试这个特性，可以在运行`caddy`时加上 `-quic` 标记：
```
$ caddy -quic
```
这样执行之后，则带有TLS加密的Web服务，在客户端支持QUIC时，将默认通过QUIC协议来完成数据的传输。

不启用QUIC时，在启动caddy之后，在服务器端查看已打开的端口号：
```
# lsof -i -P
COMMAND     PID  USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
AliYunDun  1120  root   10u  IPv4 2023899      0t0  TCP 139.196.224.72:40309->106.11.68.13:80 (ESTABLISHED)
. . . . . .
caddy      6163  root    6u  IPv6 2338478      0t0  TCP *:80 (LISTEN)
caddy      6163  root    8u  IPv6 2338479      0t0  TCP *:443 (LISTEN)
. . . . . .
```
而在通过如下命令：
```
# nohup ./caddy -quic &
```
启用QUIC提供Web服务之后，在服务器端查看已打开端口号，则可以看到如下内容：
```
# lsof -i -P
COMMAND     PID  USER   FD   TYPE  DEVICE SIZE/OFF NODE NAME
AliYunDun  1120  root   10u  IPv4 2023899      0t0  TCP 139.196.224.72:40309->106.11.68.13:80 (ESTABLISHED)
. . . . . .
caddy      6222  root    6u  IPv6 2338880      0t0  TCP *:80 (LISTEN)
caddy      6222  root    8u  IPv6 2338881      0t0  TCP *:443 (LISTEN)
caddy      6222  root    9u  IPv6 2338883      0t0  UDP *:80 
caddy      6222  root   10u  IPv6 2338885      0t0  UDP *:443
. . . . . .
```
Caddy 除了监听http的TCP 80端口和https 的TCP 443端口之外，还监听了UDP的80和443端口。

## 客户端支持

Chrome 52+ 支持QUIC而无需白名单，但需要确认 **#enable-quic** 标记已经被启用了。通过在Chrome浏览器的地址栏输入`chrome://flags/`：

![Enable QUIC](https://www.wolfcstech.com/images/1315506-9961dddafc3736d1.png)

并根据需要启用QUIC。

然后通过Chrome打开你的网站，则它应该是以QUIC提供服务的！可以通过打开inspector 工具并进入Security tab来验证这一点。重新加载页面并点击来查看连接详情：

![caddy005.png](https://www.wolfcstech.com/images/1315506-b8ea9d1418be9780.png)

如果你使用老版的Chrome，则为了省事，可以升级一下。

如果你不想升级，则可以：你将需要以特殊的参数来运行Chrome。再Mac上 (将YOUR_SITE替换为你的网站的实际域名)执行如下命令：
```
$ /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
    --user-data-dir=/tmp/chrome \
    --no-proxy-server \
    --enable-quic \
    --quic-host-whitelist="YOUR_SITE" "YOUR_SITE"
```

# QUIC的好处
QUIC是基于UDP的TLS+HTTP的可靠传输协议。它加速了TLS握手为只有一个往返，避免了TCP慢启动，并提供了网络切换时的可靠性。通过QUIC可以让网站加载更快且更可靠。

# 问题排解
首先，确保在Caddyfile文件中为域名做了适当的设置，还要确保在启动Chrome的命令行中为域名做了适当的设置。

接着，网站必须使用一个真实的可信的证书（至少，是在写的时候）。

如果那都是好的，而且你对Go语言比较了解，则你可以添加 `import "github.com/lucas-clemente/quic-go/utils"`，并在Caddy的`main()`函数的某个地方调用`utils.SetLogLevel(utils.LogLevelDebug)`。那将提供非常详细的输出。（注意这个log设施不是一个公共的API）。

当你进入`chrome://net-internals/#events`，你应该看到一些QUIC事件被标为红色。

![Net Events](https://www.wolfcstech.com/images/1315506-f9ebb69925a740ef.png)
