---
title: Netty HTTP on Android
date: 2016-11-13 17:46:49
categories: Android开发
tags:
- Android开发
- Java开发
---

Netty是一个NIO的客户端服务器框架，它使我们可以快速而简单地开发网络应用程序，比如协议服务器和客户端。它大大简化了网络编程，比如TCP和UDP socket服务器。

<!--more-->

“快速而简单”并不意味着开发出来的应用可维护性或性能不好。Netty已经实现了大量的协议，比如FTP，SMTP，HTTP，以及各种基于二进制和文本的传统协议。可以说Netty已经找到了一种方法来实现简单的开发，高性能，稳定性，灵活性而不需要做妥协。

Netty的结构大体如下图这样：

![Netty Structure](../images/1315506-317c8abe3082b0bb.png)

就设计而言，Netty给不同的传输类型，不管是阻塞的还是非阻塞的，提供了统一的接口。它基于一个灵活的和可扩展的事件模型，这使得处理不同逻辑的部分可以有效的隔离开来。它具有高度可定制的线程模型 - 单线程，一个或多个线程池，比如SEDA。它还提供无连接的datagram socket支持。

如Netty这般，功能如此强大，性能如此优良的网络库，不用在Android上真是可惜了。这里我们就尝试将Netty用在Android上。

# 下载Netty

首先是下载Netty。[Netty的官网地址](http://netty.io/)，我们可以在这里找到下载Netty的地址，当然还有许许多多的文档。这里使用了当前4.1.x最新版的Netty 4.1.4，下载地址：

```
http://dl.bintray.com/netty/downloads/netty-4.1.4.Final.tar.bz2
```

解压之后，为了省事，直接将netty-4.1.4.Final/jar/all-in-one/下的netty-all-4.1.4.Final.jar拷贝进了工程下app module的libs目录下。

# Netty的简单使用

不出意外，直接通过编译。接着我们就参考`netty/example/src/main/java/io/netty/example/http/snoop`中client部分的代码，将Netty用起来，这主要有如下几个类：
```
package io.netty.example.http.snoop;

import java.net.URI;
import java.net.URISyntaxException;

import javax.net.ssl.SSLException;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.Channel;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.http.DefaultFullHttpRequest;
import io.netty.handler.codec.http.HttpHeaderNames;
import io.netty.handler.codec.http.HttpHeaderValues;
import io.netty.handler.codec.http.HttpMethod;
import io.netty.handler.codec.http.HttpVersion;
import io.netty.handler.ssl.SslContext;
import io.netty.handler.ssl.SslContextBuilder;
import io.netty.handler.ssl.util.InsecureTrustManagerFactory;

public class HttpClient {
    public static final String TAG = "NettyClient";

    public static void getResponse(final String url) {
        new Thread() {
            @Override
            public void run() {
                try {
                    URI uri = new URI(url);
                    String scheme = uri.getScheme() == null ? "http" : uri.getScheme();
                    String host = uri.getHost();
                    int port = uri.getPort();

                    if (port == -1) {
                        if ("http".equalsIgnoreCase(scheme)) {
                            port = 80;
                        } else if ("https".equalsIgnoreCase(scheme)) {
                            port = 443;
                        }
                    }
                    if (!"http".equalsIgnoreCase(scheme) && !"https".equalsIgnoreCase(scheme)) {
                        System.err.println("Only HTTP(S) is supported.");
                        return;
                    }

                    final boolean ssl = "https".equalsIgnoreCase(scheme);
                    final SslContext sslCtx;
                    if (ssl) {
                        sslCtx = SslContextBuilder.forClient().trustManager(InsecureTrustManagerFactory.INSTANCE).build();
                    } else {
                        sslCtx = null;
                    }

                    EventLoopGroup group = new NioEventLoopGroup();
                    try {
                        Bootstrap bootstrap = new Bootstrap();
                        bootstrap.group(group).channel(NioSocketChannel.class)
                                .handler(new HttpClientInitializer(sslCtx));

                        // Make the connection attempt.
                        Channel channel = bootstrap.connect(host, port).sync().channel();

                        // Prepare the HTTP request.
                        DefaultFullHttpRequest request = new DefaultFullHttpRequest(HttpVersion.HTTP_1_1,
                                HttpMethod.GET, url);
                        request.headers().set(HttpHeaderNames.HOST, host);
                        request.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderNames.KEEP_ALIVE);
                        request.headers().set(HttpHeaderNames.ACCEPT_ENCODING, HttpHeaderValues.GZIP);

                        // Send the HTTP request.
                        channel.writeAndFlush(request);

                        // Wait for the server to close the connection.
                        channel.closeFuture().sync();
                    } finally {
                        group.shutdownGracefully();
                    }
                } catch (URISyntaxException e) {
                } catch (SSLException e) {
                } catch (InterruptedException e) {
                }
            }
        }.start();
    }
}
```
这个class提供给外部调用的接口。使用者可以传入URL，将借由这个类，通过Netty来访问网络并获取响应。然后来看HttpClientInitializer：
```
package io.netty.example.http.snoop;

import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.http.HttpClientCodec;
import io.netty.handler.codec.http.HttpContentDecompressor;
import io.netty.handler.ssl.SslContext;

public class HttpClientInitializer extends ChannelInitializer<SocketChannel> {

    private final SslContext sslCtx;

    public HttpClientInitializer(SslContext sslCtx) {
        this.sslCtx = sslCtx;
    }

    @Override
    public void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();

        // Enable HTTPS if necessary.
        if (sslCtx != null) {
            p.addLast(sslCtx.newHandler(ch.alloc()));
        }

        p.addLast(new HttpClientCodec());

        // Remove the following line if you don't want automatic content decompression.
        p.addLast(new HttpContentDecompressor());

        // Uncomment the following line if you don't want to handle HttpContents.
        //p.addLast(new HttpObjectAggregator(1048576));

        p.addLast(new HttpClientHandler());
    }
}
```
这个class负责对Channel的Pipeline进行初始化，这其中最关键的是HttpClientHandler：
```
package io.netty.example.http.snoop;

import android.util.Log;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.HttpContent;
import io.netty.handler.codec.http.HttpObject;
import io.netty.handler.codec.http.HttpResponse;
import io.netty.handler.codec.http.LastHttpContent;
import io.netty.util.CharsetUtil;

public class HttpClientHandler extends SimpleChannelInboundHandler<HttpObject> {

    @Override
    public void channelRead0(ChannelHandlerContext ctx, HttpObject msg) {
        if (msg instanceof HttpResponse) {
            HttpResponse response = (HttpResponse) msg;

            Log.i(HttpClient.TAG, "STATUS: " + response.status());
            Log.i(HttpClient.TAG, "VERSION: " + response.protocolVersion());

            if (!response.headers().isEmpty()) {
                for (CharSequence name: response.headers().names()) {
                    for (CharSequence value: response.headers().getAll(name)) {
                        Log.i(HttpClient.TAG, "HEADER: " + name + " = " + value);
                    }
                }
            }
        }
        if (msg instanceof HttpContent) {
            HttpContent content = (HttpContent) msg;
            String responseContent = content.content().toString(CharsetUtil.UTF_8);
            Log.i(HttpClient.TAG, responseContent);

            if (content instanceof LastHttpContent) {
                ctx.close();
            }
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```
我们正是通过实现HttpClientHandler，而获取到Netty返回给我们的响应的。

在我们的Android应用代码中调用`HttpClient`来通过Netty从网络获取响应：
```
package io.netty.example.http.snoop;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";

    private TextView mTextScreen;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button btnGetIpInfo = (Button) findViewById(R.id.btn_get_ip_info_with_netty);
        btnGetIpInfo.setOnClickListener(mBtnClickListener);

        mTextScreen = (TextView) findViewById(R.id.text_screen);
    }

    View.OnClickListener mBtnClickListener = new View.OnClickListener() {

        @Override
        public void onClick(View v) {
            String url = "http://ip.taobao.com/service/getIpInfo.php?ip=123.58.191.68";
            mTextScreen.setText("To access " + url);
            Log.i(TAG, "To access " + url);
            if (R.id.btn_get_ip_info_with_netty == v.getId()) {
                HttpClient.getResponse(url);
            }
        }
    };
}
```
当然不能忘记了在AndroidManifest.xml中添加对INTERNET权限的请求：
```
    <uses-permission android:name="android.permission.INTERNET"/>
```
做完了所有这些之后，Netty基本上就可以跑起来了。不过意外还是发生了：
```
08-10 10:13:33.670 17720-17720/io.netty.example.http.snoop6.myapplication I/MainActivity: To access http://ip.taobao.com/service/getIpInfo.php?ip=123.58.191.68
08-10 10:13:33.818 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err: java.lang.NoClassDefFoundError: com.jcraft.jzlib.Inflater
08-10 10:13:33.818 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.compression.JZlibDecoder.<init>(JZlibDecoder.java:27)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.compression.ZlibCodecFactory.newZlibDecoder(ZlibCodecFactory.java:122)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.http.HttpContentDecompressor.newContentDecoder(HttpContentDecompressor.java:57)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.http.HttpContentDecoder.decode(HttpContentDecoder.java:87)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.http.HttpContentDecoder.decode(HttpContentDecoder.java:46)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.MessageToMessageDecoder.channelRead(MessageToMessageDecoder.java:88)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:372)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:358)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:350)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.CombinedChannelDuplexHandler$DelegatingChannelHandlerContext.fireChannelRead(CombinedChannelDuplexHandler.java:435)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.ByteToMessageDecoder.fireChannelRead(ByteToMessageDecoder.java:293)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.ByteToMessageDecoder.fireChannelRead(ByteToMessageDecoder.java:280)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:396)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.handler.codec.ByteToMessageDecoder.channelRead(ByteToMessageDecoder.java:248)
08-10 10:13:33.826 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.CombinedChannelDuplexHandler.channelRead(CombinedChannelDuplexHandler.java:250)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:372)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:358)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:350)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1334)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:372)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:358)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:926)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:129)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:571)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.nio.NioEventLoop.processSelectedKeysPlain(NioEventLoop.java:474)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:428)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:398)
08-10 10:13:33.834 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:877)
08-10 10:13:33.841 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:144)
08-10 10:13:33.841 17720-18708/io.netty.example.http.snoop6.myapplication W/System.err:     at java.lang.Thread.run(Thread.java:841)
08-10 10:13:33.841 17720-18708/io.netty.example.http.snoop6.myapplication I/NettyClient: --------- beginning of /dev/log/system
08-10 10:35:02.162 17720-17720/io.netty.example.http.snoop6.myapplication I/Timeline: Timeline: Activity_idle id: android.os.BinderProxy@41c10f28 time:1282964865
```
有一个class `com.jcraft.jzlib.Inflater`找不到。这还需要添加对jzlib的依赖：
```
    compile 'com.jcraft:jzlib:1.1.2'
```
至此顺利地将Netty跑起来：
```
{
    "code":0,
    "data":{
        "country":"中国",
        "country_id":"CN",
        "area":"华东",
        "area_id":"300000",
        "region":"浙江省",
        "region_id":"330000",
        "city":"杭州市",
        "city_id":"330100",
        "county":"",
        "county_id":"-1",
        "isp":"网易网络",
        "isp_id":"1000119",
        "ip":"123.58.191.68"
    }
}
```
# Netty的裁剪
Netty很强大，all-in-one jar用起来很方便，这很不错。但all-in-one jar有点大，其中包含的一些诸如对memcache，redis，stomp，sctp和udt等的支持，我们在移动端并不会用掉。因而需要对它做一点裁剪。

既然不能用all-in-one包，那就把需要的几个jar文件单独copy进我们的工程好了。对于android而言，目测netty-4.1.4.Final/jar/下我们需要拷贝的jar文件主要有下面这些：
```
netty-codec-4.1.4.Final.jar
netty-codec-http-4.1.4.Final.jar
netty-codec-http2-4.1.4.Final.jar
netty-codec-socks-4.1.4.Final.jar
netty-common-4.1.4.Final.jar
netty-handler-4.1.4.Final.jar
netty-transport-4.1.4.Final.jar
```
用这些文件来替换之前的netty-all-4.1.4.Final.jar。遇到了编译错误：
```
:app:compileDebugJavaWithJavac
Full recompilation is required because at least one of the classes of removed jar 'netty-all-4.1.4.Final.jar' requires it. Analysis took 0.364 secs.
/media/data/MyProjects/MyApplication/app/src/main/java/io/netty/example/http/snoop/myapplication/HttpClient.java:67: 错误: 无法访问ByteBufHolder
                        request.headers().set(HttpHeaderNames.HOST, host);
                               ^
  找不到io.netty.buffer.ByteBufHolder的类文件
/media/data/MyProjects/MyApplication/app/src/main/java/io/netty/example/http/snoop/myapplication/HttpClientInitializer.java:39: 错误: 无法访问ByteBufAllocator
            p.addLast(sslCtx.newHandler(ch.alloc()));
                                                ^
  找不到io.netty.buffer.ByteBufAllocator的类文件
/media/data/MyProjects/MyApplication/app/src/main/java/io/netty/example/http/snoop/myapplication/HttpClientHandler.java:48: 错误: 找不到符号
            String responseContent = content.content().toString(CharsetUtil.UTF_8);
                                            ^
  符号:   方法 content()
  位置: 类型为HttpContent的变量 content
注: /media/data/MyProjects/MyApplication/app/src/main/java/io/netty/example/http/snoop/myapplication/HttpClient.java使用或覆盖了已过时的 API。
注: 有关详细信息, 请使用 -Xlint:deprecation 重新编译。
3 个错误
:app:compileDebugJavaWithJavac FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:compileDebugJavaWithJavac'.
> Compilation failed; see the compiler error output for details.

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED
```
看来是少了一些东西了，`ByteBufHolder`。那就把`netty-buffer-4.1.4.Final.jar`也加进工程里。再次编译，继续出错，这是在产生APK时遇到了麻烦：
```
:app:transformResourcesWithMergeJavaResForDebug FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:transformResourcesWithMergeJavaResForDebug'.
> com.android.build.api.transform.TransformException: com.android.builder.packaging.DuplicateFileException: Duplicate files copied in APK META-INF/INDEX.LIST
  	File1: /media/data/MyProjects/MyApplication/app/libs/netty-codec-4.1.4.Final.jar
  	File2: /media/data/MyProjects/MyApplication/app/libs/netty-transport-4.1.4.Final.jar
  	File3: /media/data/MyProjects/MyApplication/app/libs/netty-buffer-4.1.4.Final.jar
  	File4: /media/data/MyProjects/MyApplication/app/libs/netty-codec-socks-4.1.4.Final.jar
  	File5: /media/data/MyProjects/MyApplication/app/libs/netty-handler-4.1.4.Final.jar
  	File6: /media/data/MyProjects/MyApplication/app/libs/netty-codec-http2-4.1.4.Final.jar
  	File7: /media/data/MyProjects/MyApplication/app/libs/netty-codec-http-4.1.4.Final.jar


* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 15.352 secs
Duplicate files copied in APK META-INF/INDEX.LIST
	File1: /media/data/MyProjects/MyApplication/app/libs/netty-codec-4.1.4.Final.jar
	File2: /media/data/MyProjects/MyApplication/app/libs/netty-transport-4.1.4.Final.jar
	File3: /media/data/MyProjects/MyApplication/app/libs/netty-buffer-4.1.4.Final.jar
	File4: /media/data/MyProjects/MyApplication/app/libs/netty-codec-socks-4.1.4.Final.jar
	File5: /media/data/MyProjects/MyApplication/app/libs/netty-handler-4.1.4.Final.jar
	File6: /media/data/MyProjects/MyApplication/app/libs/netty-codec-http2-4.1.4.Final.jar
	File7: /media/data/MyProjects/MyApplication/app/libs/netty-codec-http-4.1.4.Final.jar

11:03:39: External task execution finished 'assembleDebug'.
```
总是报`Duplicate files copied in APK META-INF/INDEX.LIST`的错误。这主要是因为这些jar文件里，不同的jar文件中的`META-INF/INDEX.LIST`包含了相同的内容所致。这需要在`build.gradle`中的android元素里添加如下的配置：
```
    packagingOptions {
        exclude 'META-INF/INDEX.LIST'
    }
```
再次编译，这次则是包`Duplicate files copied in APK META-INF/io.netty.versions.properties`。这证明了前面的方法是行之有效的，于是把`META-INF/io.netty.versions.properties`也加进`packagingOptions`的exclude列表：
```
    packagingOptions {
        exclude 'META-INF/INDEX.LIST'
        exclude 'META-INF/io.netty.versions.properties'
    }
```
再次编译，编译通过。但是在运行的时候遇到了一点小麻烦：
```
11:29:59.685 9005-9050/com.example.hanpfei0306.myapplication W/dalvikvm: threadid=11: thread exiting with uncaught exception (group=0x41959ce0)
08-11 11:29:59.685 9005-9050/com.example.hanpfei0306.myapplication E/AndroidRuntime: FATAL EXCEPTION: Thread-1197
                                                                                     Process: com.example.hanpfei0306.myapplication, PID: 9005
                                                                                     java.lang.NoClassDefFoundError: io.netty.resolver.DefaultAddressResolverGroup
                                                                                         at io.netty.bootstrap.Bootstrap.<clinit>(Bootstrap.java:53)
                                                                                         at com.example.hanpfei0306.myapplication.HttpClient$1.run(HttpClient.java:57)
```
提示找不到class io.netty.resolver.DefaultAddressResolverGroup，看来我们的jar文件是加少了，把`netty-resolver-4.1.4.Final.jar`也加进来。终于，我们前面编写的HttpClient能够正常地跑起来了。经过一番裁剪，Netty的大小大概从3.4MB减小到2.9MB。

总结一下，若不用体型巨大的`netty-all` jar文件，则我们需要导入如下的这些jar文件以编译和运行Netty：
```
netty-buffer-4.1.4.Final.jar
netty-codec-4.1.4.Final.jar
netty-codec-http-4.1.4.Final.jar
netty-codec-http2-4.1.4.Final.jar
netty-codec-socks-4.1.4.Final.jar
netty-common-4.1.4.Final.jar
netty-handler-4.1.4.Final.jar
netty-resolver-4.1.4.Final.jar
netty-transport-4.1.4.Final.jar
```

# TODO
Netty的诸多抽象，比如Bootstrap，Channel，EventLoopGroup，Handler，codec，Buffer等诸多高级特性，以及它的NIO接口，这里都没有涉及，线程模型也没有仔细厘清。这里只是最最简单的一个使用范例，要把Netty很好地应用在实际的项目中，还需要对Netty本身更深入的研究。

同时，对Netty的裁剪可能也过于粗糙，或许还有更多的东西可以裁剪掉，以减小最终的APP的大小。

要使用Netty来支持HTTP/2也还需要做更多的事情。

但窥探到Netty的灵活强大，还是让我们对这个库充满期待。

# 参考文档

[多个jar包的合并](http://www.cnblogs.com/029zz010buct/p/4882568.html)

[Netty4.x中文教程系列](http://www.cnblogs.com/zou90512/p/3492287.html)

[netty-4-user-guide](https://github.com/waylau/netty-4-user-guide)
