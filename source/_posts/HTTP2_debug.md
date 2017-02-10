---
title: HTTP/2 流量调试
date: 2016-11-18 14:35:49
tags: 
- HTTP/2
- 网络
---

当前主要可以通过浏览器和Wireshark等工具调试HTTP/2流量。

# 使用浏览器调试HTTP/2流量

HTTP/2 引入二进制分帧层（Binary Framing），将每个请求和响应分割为更小的帧，并对它们进行二进制编码。与此同时，HTTP/2 沿用之前 HTTP/1.1中的绝大部分语义，上层应用基本上感知不到 HTTP/2 的存在。通过浏览器提供的网络调试工具我们可以清晰地看到请求和响应的详细信息。

<!--more-->

对于Chromium浏览器，打开"更多工具(L) -> 开发者工具(D)"，然后选中"Network"，并选中要查看的文件，就可以清晰地看到HTTP/2请求及响应的信息了。如：

![](https://www.wolfcstech.com/images/1315506-af5d812ebf583fa3.png)
从上图可以看到，HTTP/2资源的内容，与HTTP/1.1资源的内容相比，只有一些细微的变化，如所有的头部字段名都是小写的，引入了以":"开头的伪首部字段等。

然而，浏览器的调试工具只能看到请求和响应的信息。对于连接建立、数据传输，及流控这样的过程信息，浏览器的调试工具就显得无能为力了。这些更细节的信息可以通过Wireshark来看。

# 使用 Wireshark 调试 HTTP/2 流量

尽管HTTP/2规范并没有严格要求只能基于TLS传输HTTP/2流量，然而目前基于TLS传输HTTP/2已然成为事实的标准了。绝大部分提供HTTP/2服务器的网站都基于TLS来提供，常见的客户端浏览器，如Chrome，Firefox更是完全不提供对明文传输的HTTP/2的支持。因而通过Wireshark调试HTTP/2流量最主要的即是解密HTTPS流量，进而借助Wireshark对HTTP/2数据的解析，看到客户端与服务器之间详细的帧交互过程。

Wireshark 的抓包原理是直接读取并分析网卡数据，要想让它解密 HTTPS 流量，有两个办法：1）如果拥有 HTTPS 网站的加密私钥，可以用来解密这个网站的加密流量；2）某些浏览器支持将 TLS 会话中使用的对称加密密钥保存在外部文件中，可供 Wireshark 解密之用。

## 设置`SSLKEYLOGFILE`环境变量解密HTTPS流量

Firefox 和 Chrome 都支持生成上述第二种解密方式所需的文件，具体格式见 [NSS Key Log Format](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/NSS/Key_Log_Format)。但只有设置了系统环境变量`SSLKEYLOGFILE`，Firefox 和 Chrome 才会生成该文件，它们将这个系统变量的值作为文件保存的路径。先来加上这个环境变量（Ubuntu）：
```
$ mkdir ~/sslkeylogfile && touch ~/sslkeylogfile/keylogfile.log

$ echo -e '\nexport SSLKEYLOGFILE=~/sslkeylogfile/keylogfile.log' >> ~/.bash_profile && . ~/.bash_profile
```
接着，通过选择`Edit -> Preferences... -> Protocols -> SSL`打开Wireshark 的 SSL 配置面板(Ubunut版，Mac版通过`Wireshark -> Preferences...`打开首选项)，在「(Pre)-Master-Secret log filename」选项中选择`SSLKEYLOGFILE`文件，或输入该文件的路径。如下图：

![ssl_config.png](https://www.wolfcstech.com/images/1315506-20425d67971ec92b.png)
最好将「SSL debug file」也配上，这样解密过程中的日志都会记录下来，便于调试分析。

通过 **终端** 启动 Firefox 或 Chrome（**确保这些浏览器能读取到环境变量**）：
```
$ chromium-browser
```
这时再访问 HTTPS 网站，`sslkeylog.log`文件中应该包含浏览器写入的数据。如下面这样：
```
$ cat ~/sslkeylogfile/keylogfile.log

CLIENT_RANDOM 24103980248f266808400da7e48058f6c779b4b98fe8cae45981bafe7502b8ff 5e7e3a526fbf8c5a3567758f6aa412ea6c36c52d98e9a3dfde1d16f09714551bc64f62ca3cd9df2448b12dbf995d0b14
CLIENT_RANDOM 31c9cac267465543d6ea1ea358dd41fc9ed02aa8af45b0238b82e50915bc3360 dadf649c846b125dad5b1215e9d45ee34ba5241ecbc4856820ea7843177dc424a575fd788f007241c0252c718426dd92
CLIENT_RANDOM 36b8759c3bbb3df2645382240384675f1615f4a432ca7c58bf142776f752ee05 365c321e8d91834da7e8d44cbb963b2bc1b90f100a50fb7831bd5b9b7ec62d94c9a6de99da78c3c6aac93568cc675113
CLIENT_RANDOM 24a461fc4ae2295b81dc4eab6aff2e520e0fc2623091b249f59bfb46b1c3567d e2c8b46febbb61f0fa3e6a6857cd27fe95aa1a7616a522115488f35f30c3092d05b295cba99429d7c16d857cfb87953e
CLIENT_RANDOM dcc1b0688d4dd99b3b5a4cacccfb9fa5d4861c439385abfb37cd4f98ed212091 3c6bb8767ddb4e6797880aadf5f59f39edaf15bc17c7a3632603803bffec45239261b9ae87d42495952c86c65158dbb4
CLIENT_RANDOM c96b42c46769d87fc06884da879a2f274047356ee3516f944aa0f37f72bbed0e 758e59ee207ff1667ee6643eb31d62aacb4a3fde3e39ab0654a003b0b015acd7e53de41ea754255e4bade1f25897643a
CLIENT_RANDOM 766c0a2f84030be01513ffff1c36bff61872f0a8e039a40694eeee5b3a65c4ca efbdbc6cafdd32891df40daaa8d2c4778db889512c4e6ccd8a15a7f1e34ab55d7a657021eb9954883dc320cc632e8755
CLIENT_RANDOM b91cbe5fb40149006dbf95a74f1a8f9f0999a78c27dae42d169ca50521477c2c b3e9963f25598e9841ae58857ce13a891c22275f7f2d5c260446cba7355ccdcf41eeabe8f050350078ffe7a16a7c5219
CLIENT_RANDOM de8e0dbdc518459724cf5ed3cc09802c96fa558abe7c95ec4f0748700bc3e229 0cda6e9665758c54819cc00a09eb5f4249eb33519a16ae62911ffff090539aebd3138ebb4ee0ba7cab6630b3e255fffc
CLIENT_RANDOM 6798fa4e2396f700f6309d55721badf73081ed2a8066db2f7bf652245c61076c 025a4679f2d399433a7dc9604f1b1a970dcc54b6da5166a188b061ece87cac1919e7b56be8455cf24ddffe1083694bb1
CLIENT_RANDOM d89d9e38c21af551d816f451ac3b25d29899f5623fa375f7c55897c06310e784 e9d56f1b084327850c96e30fa87488e7d83fcca7e5cd2200876aa8354d841d5432f339db04041fda5f79202c528e5399
CLIENT_RANDOM 5db78a4b93747b795867ed5800ed2f0c763d74f40fd9859e358689efcc3082c3 3e8edf030deae2d2ff78858b96d7525527e4bbdffe4caf62014a1977f008a7a9a6e96f08e07624a48ad45d45a3ef13c5
CLIENT_RANDOM 2902990ba00c5ad72fddf9a82fd78bdc40e347d423f5aaff71956780683b4df7 f51e107ad5db85ff16da162abb050482b196ffa0f99c2c1e834b73dd46454d322b53a85c14e5a266e05be20d19b1fd2a
......
```
检查无误后，就可以开启 Wireshark，选择合适的网卡开始抓包。为了减少不必要的数据包对我们分析的干扰，我们可以只针对某个域名的TCP 443端口来抓包，以减少抓取的数据包的数量，如：

![Capture Filter](https://www.wolfcstech.com/images/1315506-c7a4380946f99370.png)
新版 Wireshark (如我目前在用的Version 2.0.2) 在配置了 TLS 解密后，会自动识别并解析 HTTP/2 流量。访问想要抓包的 HTTP/2 网站，就可以轻松看到想要的 HTTP/2 数据包了。如下图：

![wireshark_http2.png](https://www.wolfcstech.com/images/1315506-76a735cc794a4722.png)
这种方法也可以用在解密使用 HTTP/1 的 HTTPS 网站上。

每次都要从终端启动浏览器还是挺麻烦的，我们也可以为GUI设置环境变量，以便于我们不在命令行终端启动浏览器也可以解密HTTPS流量。方法是：
```
$ sudo echo -e '\nexport SSLKEYLOGFILE=~/sslkeylogfile/keylogfile.log' >> /etc/profile
```
也可以通过其它任何个人偏好的编辑器为`/etc/profile`添加`export`那一行。更新了`/etc/profile`之后，注销当前登录并重新登录。在GUI中启动chromium浏览器，也可以通过Wireshark解密HTTPS流量了。不过要注意，这个设置对所有用户均全局生效。为了系统安全最好在利用Wireshark调试HTTPS流量之后移除这一设置。

Wireshark就像一副眼镜一样，戴上它之后，可以让我们对网络世界发生的一切看得更加清楚，然而即使有了这样强大好用的工具，要想对协议有真正的理解，仔细地阅读协议规范文件也是必不可少的，关于HTTP/2协议的细节，可以参考[HTTP2规范（RFC7540）](https://www.wolfcstech.com/2016/10/29/http2-spec/)、[HPACK：HTTP/2的首部压缩 (RFC7541)](https://www.wolfcstech.com/2016/10/29/hpack-spec/)和[应用层协议协商（ALPN）规范（RFC7301）](https://www.wolfcstech.com/2016/10/29/alpn-spec/)。

## 导入服务器私钥解密HTTPS流量

还可以通过为Wireshark导入服务器RSA私钥来解密HTTPS流量。方法是选择`Edit -> Preferences... -> Protocols -> SSL`打开Wireshark 的 SSL 配置面板，点击"RSA keys list"后的"Edit..."按钮：

![rsa_key_record.png](https://www.wolfcstech.com/images/1315506-478ea8593495d286.png)
添加一个RSA私钥项，输入适当的IP地址，端口，解密后的协议，及私钥文件的路径，其中私钥必须存为 PEM 格式，这种格式的私钥文件类似于下面这样：
```
-----BEGIN RSA PRIVATE KEY-----
MIIJKQIBAAKCAgEAw5JweP8ACiLHFAs9syY0yzdjJh3WQntcq1HM9xJXJrHRT1+n
Als88+bh7HxmYbrYXtZhYEjW00wjmlB1Sy0biX17VS8SY7Ssx7XRRJXtovSd7nc8
KwZI2Rgh0e7PXCatwhEonCdlEZTEHQ9eTE8Acc9dH0BLml47sEJFxWgBYinCkrWt
jRMR234CZRRqXUtmFgjUq5DcBignzYTAtzAi/1RxwjuNfYtSw6jx7mNCKt1Np3Bm
....... some 20-100 lines of base64 encoded data ...............
+TtJwCHfBRGIyI3hGA1eBa8Lc3nhXPjUwtp7q+2gwpsq/25gt2tiUfSak7frhFal
9gP00/d9nJXwh2UR3G/oEsXWORfRUVAgjyzhXrrJ+uQJBgUVEZ1D0lI6qfdc
-----END RSA PRIVATE KEY-----
```
对于nginx服务器来说，通常通过`ssl_certificate_key`项配置的是站点私钥，这个私钥通常是在申请证书时，通过类似于如下这样的方式产生：
```
$ openssl genrsa 4096 > domain.key
```
关于私钥格式的更多信息，可以参考[Wireshark的SSL Wiki](https://wiki.wireshark.org/SSL#Wireshark)。

但这种方法对客户端与服务器协商的加密套件有要求。如果加密套件的密钥交换算法是ECDHE，也就是当前大多数HTTPS流量所选择的算法，则解密HTTPS流量将失败。可以通过Wireshark抓取的TLS握手的`Server Hello`消息来查看客户端与服务器协商的加密套件：

![Server Hello.png](https://www.wolfcstech.com/images/1315506-1fc6886778b7c893.png)
客户端与服务器协商加密套件的过程大体为，客户端在其TLS握手的`Client Hello`消息的`Cipher Suites`字段中发送自己支持的加密套件。如通过curl访问http2资源：
```
$ curl --http2 -v https://www.wolfcstech.com/
```
可以看到curl支持的加密套件集：
![client_cipher_suites.png](https://www.wolfcstech.com/images/1315506-9762724006f3c3d2.png)
对于nginx，我们通过`ssl_ciphers`选项为其配置加密套件，如：
```
	 	# https://github.com/cloudflare/sslconfig/blob/master/conf
		ssl_ciphers                EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
		ssl_prefer_server_ciphers  on;
```
加密套件以":"分隔。服务器从为其配置的加密套件集中选择排序最靠前的客户端支持的加密套件。对于上面的服务器加密套件集配置，用curl访问服务器，协商出来的加密套件为`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)`，这种加密套件的密钥交换算法正是ECDHE，也即不支持Wireshark加密的算法。

我们可以通过修改服务器的配置，以便Wireshark可以解密HTTPS流量。修改之后nginx服务器的加密套件配置为：
```
	 	# https://github.com/cloudflare/sslconfig/blob/master/conf
		ssl_ciphers                EECDH+CHACHA20:EECDH+CHACHA20-draft:RSA+AES128:EECDH+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
		ssl_prefer_server_ciphers  on;
```
与前面那个配置相比，仅有的改动是对调了`RSA+AES128`和`EECDH+AES128`两个加密套件的位置。再次通过curl访问服务器，可以看到HTTP2流量被解密了：
![curl_http2.png](https://www.wolfcstech.com/images/1315506-88cc08373bee2950.png)

这次协商的加密套件为`TLS_RSA_WITH_AES_128_GCM_SHA256 (0x009c)`，其密钥加密算法为RSA。

不过此时通过chrome浏览器访问网站，就会发现页面已经打不开了，如下图所示：
![cipher_suite_config_error_for_http2.png](https://www.wolfcstech.com/images/1315506-5ebd6b734aa28bf5.png)
仔细看的话，可以看到`ERR_SPDY_INADEQUATE_TRANSPORT_SECURITY`。这是由于为了安全性考虑，HTTP/2规范建立了[加密套件黑名单](http://http2.github.io/http2-spec/#BadCipherSuites)，并强制要求HTTP/2服务不得配置这样的加密套件。Chromium浏览器严格遵守HTTP/2规范，且在TLS协商阶段，服务器选择了加密套件黑名单中的`TLS_RSA_WITH_AES_128_GCM_SHA256 (0x009c)`，因而连接直接被断开了。通过Wireshark抓包来看：

![http2_goaway.png](https://www.wolfcstech.com/images/1315506-2c7fc5a87c6557ec.png)
可见是HTTP2层，在与服务器建立连接之后，浏览器就立即发送`GOAWAY`帧断开了连接。

然而curl似乎并没有严格遵守HTTP/2规范。

是否可以通过为Wireshark添加RSA私钥解密HTTPS流量的决定性因素，在于客户端与服务器协商的加密套件，而不在于通过HTTPS传输的流量是HTTP/1.1的还是HTTP/2的。不过可以通过这种方法让Wireshark解密的所有加密套件都已经进了HTTP/2规范的加密套件黑名单，因而对于符合规范的HTTP/2流量传输，都是无法通过这种方法来解密流量的。

参考文档：
[SSL - The Wireshark Wiki](https://wiki.wireshark.org/SSL#Wireshark)
[Ubuntu系统环境变量详解](http://blog.csdn.net/netwalk/article/details/9455893)
[使用 Wireshark 调试 HTTP/2 流量](https://imququ.com/post/http2-traffic-in-wireshark.html)
[三种解密 HTTPS 流量的方法介绍](https://imququ.com/post/how-to-decrypt-https.html)
[为curl命令启用HTTP2支持](https://www.wolfcstech.com/2016/11/14/curl_for_HTTP2_on_ubuntu/)
