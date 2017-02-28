---
title: SpringMVC 构建 RESTful Web 服务
date: 2017-02-28 17:05:49
categories: Java开发
tags:
- 后台开发
- Java开发
---

 因工作需要和个人技术兴趣，而需要学习一下 Java Web开发。本文记录了学习如何在 IntelliJ + Tomcat + Maven + SpringMVC 环境下，开发 RESTful Web 服务的过程。
 <!--more-->
# 要构建的 Web 服务
在这份文档中，我们将经历一个构建简单的 "hello, world" 级的 Spring MVC RESTful Web服务的过程。我们构建的服务将可以通过如下 URL 的 HTTP GET 请求进行访问：
```
http://localhost:8080/greeting
```

这个接口将为我们返回 [JSON](http://spring.io/understanding/JSON) 表示的问候语：
```
{"id":1,"content":"Hello, World!"}
```

我们可以通过 query 字符串中可选的参数 `name` 来定制问候语：
```
{"id":1,"content":"Hello, User!"}
```

# 创建工程
在菜单栏选择 File -> New -> Project... 打开创建工程的窗口，如下图：
![](https://www.wolfcstech.com/images/1315506-12ac58e45b36f736.png)

然后选中左边的 `Spring` 一项，在右边选择 `Spring`，以及 `Spring` 下的 `Spring MVC`。在选中 `Spring` 下的 `Spring MVC` 时， `Web Application` 也会被自动选中。选中之后，需要等 IntelliJ 加载一会儿组件的版本信息，在选中的组件后面出现括号包着的灰色版本号时，说明版本信息加载完成了。然后点击 `Next`，进入如下界面：
![](https://www.wolfcstech.com/images/1315506-3026b355e4ef271d.png)

输入工程名称，设置工程的根目录，并点击  `Finish`。此时 IntelliJ 将帮我们自动下载所需的 Jar 文件：
![](https://www.wolfcstech.com/images/1315506-a5469d94befa6bf7.png)

下载完成之后，项目就建好了。整个项目的结构如下：
![](https://www.wolfcstech.com/images/1315506-96351e2a1a8f518b.png)

lib 下存放我们下载的 jar 文件，以及其它在项目中要用到的第三方库文件。src 用于放置我们的 Java 代码，而 web 目录则存放我们的 Web 应用的配置文件等。

# 新建包
通过 IntelliJ 菜单的 `File` -> `New` -> `Package`打开创建包的窗口：
![](https://www.wolfcstech.com/images/1315506-e927c90140d0c0cb.png)

输入包名 `com.hanpfei` 并点 `OK` 键。此时，工程的整个结构如下：
![](https://www.wolfcstech.com/images/1315506-8f28f8585e32c5d8.png)

包直接位于 `src` 目录下。

# 创建资源表示类
接着我们可以创建我们的 Web 服务了。首先考虑服务的接口。服务将处理 `/greeting` 的 `GET` 请求，在 query 串中有一个可选的 `name` 参数。`GET` 请求应该返回一个 `200 OK`响应，而在响应体中是 JSON 格式表示的问候语。它看起来应该像这样：
```
{
    "id": 1,
    "content": "Hello, World!"
}
```
其中 `id` 字段是唯一的问候语标识符，`content` 是文本表示的问候语。

我们创建一个资源表示类来建模问候语表示。提供一个含有字段，构造器，和 `id` 及 `content` 的访问器的 POJO 对象。
`src/com/hanpfei/Greeting.java`：
```
package com.hanpfei;

public class Greeting {
    private final long id;
    private final String content;

    public Greeting(long id, String content) {
        this.id = id;
        this.content = content;
    }

    public long getId() {
        return id;
    }

    public String getContent() {
        return content;
    }
}
```
如我们在下面的步骤中所见，Spring 将使用 [Jackson JSON](http://wiki.fasterxml.com/JacksonHome) 库自动地将类型 `Greeting` 的实例序列化 JSON。

接着我们创建资源控制器来提供这些问候语服务。

# 创建资源控制器
在构建 RESTful Web服务的 Spring 方法中，HTTP 请求由一个控制器处理。这些组件简单地通过 [@RestController](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/RestController.html) 注解标识，下面的 `GreetingController` 处理对 `/greeting` 的 `GET` 请求，并返回 `Greeting` 类的实例。
`src/com/hanpfei/GreetingController.java`：
```
package com.hanpfei;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.atomic.AtomicLong;

@RestController
public class GreetingController {

    private static final String template = "Hello, %s!";
    private final AtomicLong counter = new AtomicLong();

    @RequestMapping("/greeting")
    public Greeting greeting(@RequestParam(value="name", defaultValue="World") String name) {
        return new Greeting(counter.incrementAndGet(),
                String.format(template, name));
    }
}
```
这个控制器是简洁而简单的，但是在底层还是有着大量的东西。让我们一步一步地分解它。

`@RequestMapping` 注解确保对 `/greeting` 的HTTP 请求被映射到 `greeting()` 方法。上面的例子没有指定 `GET`，`PUT`，`POST`，等等，因为 `@RequestMapping` 默认映射所有的 HTTP 操作。使用 `@RequestMapping(method=GET)` 来缩窄这种映射。

`@RequestParam` 将 query 字符串参数 `name` 的值绑定到 `greeting()` 方法的 `name` 参数。这个 query 字符串参数被明确地标记为可选的（默认是 `required=true` ）：如果在请求中没有，将使用 "World" 作为 `defaultValue`。

方法体的实现创建并返回一个新的具有基于 `counter` 的下一个值的 `id` 和 `content` 属性的 `Greeting` 对象，并通过使用问候语 `template` 将给定的 `name` 格式化。

传统的 MVC 控制器和上面的 RESTful Web 服务控制器之间的一个重要不同是 HTTP 响应体创建的方式。这个 RESTful Web 服务控制器不是依赖于 [视图技术](http://spring.io/understanding/view-templates) 来执行服务器端将问候数据呈现为HTML，而是简单地填充并返回Greeting对象。对象数据将被以 JSON 的格式直接写入 HTTP 响应。

上面的代码使用 Spring 4 的新 [@RestController
](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/bind/annotation/RestController.html) 注解，它将类标记为一个控制器，其中每个方法返回一个域对象而不是一个View。它是 `@Controller` 和 `@ResponseBody` 的缩写。

`Greeting` 对象必须被转换为JSON。有赖于 Spring 的 HTTP 消息转换器支持，你无需手动地做这种转换。。由于 [Jackson 2](http://wiki.fasterxml.com/JacksonHome) 在 classpath 上，Spring 的 [MappingJackson2HttpMessageConverter
](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/http/converter/json/MappingJackson2HttpMessageConverter.html) 被自动地选择来将 `Greeting` 实例转换为 JSON。

# 使应用程序可执行
尽管将这个服务打包为传统的 [WAR](http://spring.io/understanding/WAR) 文件然后部署到一个外部应用服务器上是可能的，然而下面将演示更简单的方法，创建一个独立的应用。将所有的东西打包为一个单独的，可执行的 JAR 文件，由一个老式的 Java `main()` 方法驱动。一路上，使用 Spring 的支持将 Tomcat servlet 容器嵌入为 HTTP 运行时，而不是部署到外部实例。
`src/com/hanpfei/Application.java`：
```
package com.hanpfei;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
`@SpringBootApplication`是一个方便的注解，它将添加下面所有的东西：
* `@Configuration` 标记类为应用上下文的一个 bean 定义源。
* `@EnableAutoConfiguration` 告诉 Spring Boot 基于 classpath 设置添加 beans，其它 beans 和不同的属性设置。
* 通常你将为一个 Spring MVC app 添加 `@EnableWebMvc`，但 Spring Boot 会在 classpath 上看到 **spring-webmvc** 时自动地添加它。这将应用标记为一个 Web 应用，并激活诸如设置一个 `DispatcherServlet` 这样的重要行为。
* `@ComponentScan` 告诉 Spring 在 `com.hanpfei` 包中寻找其它组件，配置，和服务，使它可以找到控制器。

`main()` 方法使用 Spring Boot 的 `SpringApplication.run()` 方法启动一个应用。你是否注意到没有一行的XML。也没有 **web.xml** 文件。这个 Web 应用是 100% 的纯 Java应用，你不必处理配置任何管道或基础设施。

然而此时我们在 IntelliJ 中发现，找不到 `org.springframework.boot` 包下的组件。我们还需要导入如下的几个 jar 包，来使这个 Web 应用运行起来：
```
spring-boot-1.4.1.RELEASE.jar
spring-boot-autoconfigure-1.4.1.RELEASE.jar
spring-boot-starter-1.5.1.RELEASE.jar
spring-boot-starter-web-1.5.1.RELEASE.jar
```
方法是在 [Maven Repository](http://mvnrepository.com/artifact/org.springframework.boot) 找到这几组件，并下载适当的版本到 `HelloHanpfeiSpring/lib` 目录下。如果本地其它项目中已经有了这些文件，或者本地 Maven 仓库（位于 `~/.m2/repository/org/springframework/boot` ）中下载过了这些文件，也可以直接拷贝过去。

有了需要的 Jar 包之后，还需要为项目添加对这些 Jar 包的依赖。方法是通过菜单栏 File -> Project Structure... （或使用 Ctrl + Alt + Shift + S ）打开如下对话框：
![](https://www.wolfcstech.com/images/1315506-5149535e1a382cc9.png)

选中左边的 `Libraries`，点击中间一栏上面的 "+" 号，选中其中的 `Java`，在弹出的文件选择对话框中，选中 `spring-boot-1.4.1.RELEASE.jar` 和 `spring-boot-autoconfigure-1.4.1.RELEASE.jar`：
![](https://www.wolfcstech.com/images/1315506-a4b839944b2a18c7.png)

然后以同样的方法添加 `spring-boot-starter-1.5.1.RELEASE.jar` 和 `spring-boot-starter-web-1.5.1.RELEASE.jar`。引用的库的配置将如下图：
![](https://www.wolfcstech.com/images/1315506-28757566ec529750.png)

点击 `OK` 键结束添加。

## 新建 Run/Debug Configuration
为了使我们的 Web 应用可以跑起来，我们还需要创建一个 `Run/Debug Configuration`。方法为通过菜单的 Run -> Edit Configuratons ... 打开编辑配置对话框：
![](https://www.wolfcstech.com/images/1315506-ae42ae12956ab118.png)

点击右上角的 "+" 号，并在弹出的一列选项中，找到 `Spring Boot` 并选中，此时对话框将变成如下的样子：
![](https://www.wolfcstech.com/images/1315506-8016887ae8866ca6.png)

我们输入 Name，配置 Main class，最终将像下面这样：
![](https://www.wolfcstech.com/images/1315506-3fb744eec40875e9.png)

点击 OK 退出。此时我们就可以通过工具栏中的 `运行` 按钮来运行我们的 Web 应用了：
![](https://www.wolfcstech.com/images/1315506-bd00f583ac9240e1.png)

此时我们点击 `运行` 按钮，结果如下：
![](https://www.wolfcstech.com/images/1315506-2cd5f5911ee33142.png)

What a Terrible Failure！！！ Web 应用进程竟然在我们的请求还没发出的时候，就直接退出了，退出了，了，了。。。
![](https://www.wolfcstech.com/images/1315506-289a03ae54a577b0.jpg)

不着急。回想前文有提到，这样运行需要借助于 Tomcat 运行时。于是我们找到相关的几个 Jar 文件，像导入 SpringBoot 一样导入我们的工程（同样可以到 [Maven Respository](http://mvnrepository.com/search?q=tomcat-embed) 下载）：
```
tomcat-embed-core-8.5.11.jar
tomcat-embed-el-8.5.11.jar
tomcat-embed-websocket-8.5.11.jar
```

再次运行我们的 Web 应用。终于欢快地跑起来了：
![](https://www.wolfcstech.com/images/1315506-23e5dcce9ab05ec8.png)

# 测试服务
接着我们来测试一下我们新建的 Web 应用。在浏览器的地址栏中输入 [http://localhost:8080/greeting](http://localhost:8080/greeting)，我们看到了：
![](https://www.wolfcstech.com/images/1315506-e0f113b1d830af39.png)

新手定律说得好，问题总是喜欢刁难初学者，这真是一点也不错。这不又出错了么。此时在 IntelliJ 的运行控制台，我们看到了如下的错误输出：
```
严重: Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.IllegalArgumentException: No converter found for return value of type: class com.hanpfei.Greeting] with root cause
java.lang.IllegalArgumentException: No converter found for return value of type: class com.hanpfei.Greeting
	at org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor.writeWithMessageConverters(AbstractMessageConverterMethodProcessor.java:187)
	at org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.handleReturnValue(RequestResponseBodyMethodProcessor.java:174)
	at org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite.handleReturnValue(HandlerMethodReturnValueHandlerComposite.java:81)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:132)
```

提示没有 `com.hanpfei.Greeting` 类的转换器，这可让我们如何是好啊。

前面不是说，只要 `Jackson` 在 classpath 上，转 JSON 的动作就会自动完成么。不对，那可能是在我们的环境中，`Jackson` 并没有在 classpath 上。像前面的 spring-boot 和 `tomcat-embed` 那样，我们将 `Jackson` 相关的几个 jar 包导入：
```
jackson-annotations-2.8.0.jar
jackson-core-2.8.6.jar
jackson-databind-2.8.6.jar
```
在浏览器中，刷新 Url，终于看到了：
```
{"id":1,"content":"Hello, World!"}
```

提供一个 `name` query 字符串参数，[http://localhost:8080/greeting?name=hanpfei](http://localhost:8080/greeting?name=hanpfei)。注意，`content` 属性的值是如何从 "Hello, World!" 变为 "Hello, hanpfei!" 的：
```
{"id":2,"content":"Hello, hanpfei!"}
```
这个变动演示了 `GreetingController` 中安排的 `@RequestParam` 如预期般运行了。 `name` 参数已经给了一个默认的值 "World"，但总是可以被明确地通过 query 字符串覆盖。

同时注意 `id` 属性是如何从 `1` 变为 `2` 的。这证明了多个请求是运行于相同的 `GreetingController` 实例之上的，且它的 `counter` 字段在每次调用时如预期那样递增了。

# 在 Tomcat 上运行
安装配置 Tomcat 的方法可以参考 [Ubuntu 16.04 Tomcat 8安装指南](https://www.wolfcstech.com/2017/02/24/Install_Tomcat8_on_Ubuntu16.04/)。这里我们主要来看在 IntelliJ 中如何将我们的 Web 应用跑在 Tomcat上。

首先我们需要创建一个`Application Server`。在菜单栏选择 File -> Settings ... ，然后选择 Build, Execution, Deployment -> Application Servers。点击中间一栏的 "+" 加号，在弹出的选项列表中选择 `Tomcat Server`，在弹出的对话框中配置 Tomcat 主目录：
![](https://www.wolfcstech.com/images/1315506-34ea6372220ea7b0.png)

完成之后点击 "OK" 按钮。最终将像下面这样：
![](https://www.wolfcstech.com/images/1315506-d254f4f9cc836af1.png)

然后添加一个 `Run/Debug Configuration`。通过菜单栏的  Run -> Edit Configuratons ... 打开编辑配置对话框，点击右上角的 "+" 号，并在弹出的一列选项中，找到 `Tomcat Server` -> `Local` 并选中。然后输入配置名称，并适当的配置 `Application server` 为我们之前创建的配置。结果如图所示：
![](https://www.wolfcstech.com/images/1315506-4ffafab9c6383dca.png)

最下面可以看到 `Warning` 行，且左方的 `Tomcat-8.0.32` 图标上有一个错误标记，说明还没有配置完全。我们还需要将项目部署到 Tomcat 服务器中。点击 Deployment，再点击中间上方的 ”+“ 号，添加一个 Artifact，最终将像下面这样：
![](https://www.wolfcstech.com/images/1315506-32342bdc28e92f8a.png)

再点击OK，整个Tomcat配置结束。

点击工具栏中的运行按钮，就可以启动 Tomcat 了，其控制台输出将在 IDEA 下方显示。再次遭受了新手定律的困扰：在控制台输出中，我们看到 Demo Spring MVC 应用部署失败，` One or more listeners failed to start.`，但需要到应用容器的 log 中寻找关于错误的更多信息。

在控制台中还可以看到如下这样的一些输出：
```
/opt/tomcat/bin/catalina.sh run
Using CATALINA_BASE:   /home/hanpfei0306/.IntelliJIdea2016.3/system/tomcat/Unnamed_HelloHanpfeiSpring
Using CATALINA_HOME:   /opt/tomcat
Using CATALINA_TMPDIR: /opt/tomcat/temp
Using JRE_HOME:        /usr/lib/jvm/java-8-openjdk-amd64
Using CLASSPATH:       /opt/tomcat/bin/bootstrap.jar:/opt/tomcat/bin/tomcat-juli.jar
[2017-02-28 03:25:20,841] Artifact HelloHanpfeiSpring:war exploded: Server is not connected. Deploy is not available.
28-Feb-2017 15:25:21.365 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server version:        Apache Tomcat/8.0.32
28-Feb-2017 15:25:21.366 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server built:          Feb 2 2016 19:34:53 UTC
28-Feb-2017 15:25:21.366 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server number:         8.0.32.0
28-Feb-2017 15:25:21.367 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Name:               Linux
28-Feb-2017 15:25:21.367 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Version:            4.4.0-62-generic
28-Feb-2017 15:25:21.367 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Architecture:          amd64
28-Feb-2017 15:25:21.369 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Java Home:             /usr/lib/jvm/java-8-openjdk-amd64/jre
28-Feb-2017 15:25:21.369 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Version:           1.8.0_121-8u121-b13-0ubuntu1.16.04.2-b13
28-Feb-2017 15:25:21.369 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Vendor:            Oracle Corporation
28-Feb-2017 15:25:21.369 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_BASE:         /home/hanpfei0306/.IntelliJIdea2016.3/system/tomcat/Unnamed_HelloHanpfeiSpring
28-Feb-2017 15:25:21.369 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_HOME:         /opt/tomcat
28-Feb-2017 15:25:21.370 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.util.logging.config.file=/home/hanpfei0306/.IntelliJIdea2016.3/system/tomcat/Unnamed_HelloHanpfeiSpring/conf/logging.properties
28-Feb-2017 15:25:21.370 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
28-Feb-2017 15:25:21.370 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcom.sun.management.jmxremote=
```

通过控制台输出中的 `CATALINA_BASE` 定义找到 log 文件夹的位置：
```
/home/hanpfei0306/.IntelliJIdea2016.3/system/tomcat/Unnamed_HelloHanpfeiSpring/logs
```

在其中的 `localhost.2017-02-28.log` 中可以看到如下的 error log：
```
28-Feb-2017 15:17:39.814 SEVERE [RMI TCP Connection(2)-127.0.0.1] org.apache.catalina.core.StandardContext.listenerStart Error configuring application listener of class org.springframework.web.context.ContextLoaderListener
 java.lang.ClassNotFoundException: org.springframework.web.context.ContextLoaderListener
	at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1308)
	at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1142)
```
即找不到 Spring 的一个 class `org.springframework.web.context.ContextLoaderListener`。在网上搜了半天，终于找到解法，[ClassNotFoundException:ContextLoaderListener](http://blog.csdn.net/lucklq/article/details/5868496)。解法其实简单，就是将下载的 Spring 的 lib 文件夹移到项目的 `web/WEB-INF/` 下。工程的结构将像下面这样：
![](https://www.wolfcstech.com/images/1315506-befcb80e64a5e036.png)

再次尝试在 Tomcat 中运行我们的 Web应用，结果如下：
![](https://www.wolfcstech.com/images/1315506-db32dcf0f80e6d62.png)

部署终于成功了。（通过菜单栏打开 File -> Project Structure... （或使用 Ctrl + Alt + Shift + S ）工程配置对话框，选中左边的 `Libraries`，重新配置之前我们导入的 Jar 包，将仍然可以 通过前面配置的 `Spring Boot` 运行配置运行我们的 Web 应用，否则将由于符号找不到而运行失败。）

而当我们通过，却发现返回了 404：
![](https://www.wolfcstech.com/images/1315506-8976b85acb1493d7.png)

# 配置 web.xml
我们的 Web 服务没能在 Tomcat 中跑起来，不急，我们还没通过 web.xml 配置 Servlet 呢。

打开 `HelloHanpfeiSpring/web/WEB-INF` 下的 `web.xml` 文件，稍微更新一下`web.xml` 的版本，可以支持更高级的一些语法，如下：
```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
         http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <display-name>HelloHanpfeiSpring Web Application</display-name>

</web-app>
```

在 `<web-app>` 中加入一个 `Servlet`：
```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
         http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <display-name>HelloHanpfeiSpring Web Application</display-name>

    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

该 `Servlet` 名为 dispatcher（名称可修改），用于拦截请求（url-pattern为 / ，说明拦截所有请求），并交由 Spring MVC 的后台控制器来处理。这一项配置是必须的。

为了能够处理中文的 `POST` 请求，再配置一个 `encodingFilter`，以避免 `POST` 请求中文出现乱码情况：
```
    <filter>
        <filter-name>encodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>encodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
```
至此，`web.xml` 配置完毕。

# xxx-servlet.xml配置
 在配置完 `web.xml` 后，需配置与 `web.xml` 在同级目录下的 `dispatcher-servlet.xml`（ -servlet 前面的 `xxx` 是在 `Servlet` 里面定义的servlet名）。

首先加入 `component-scan` 标签，指明 `controller` 所在的包，并扫描其中的注解（最好不要复制，输入时按IDEA会在beans xmlns中添加相关内容）：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">
    
    <!--指明 controller 所在包，并扫描其中的注解-->
    <context:component-scan base-package="com.hanpfei"/>
</beans>
```

再进行js、image、css等静态资源访问的相关配置，这样，SpringMVC才能访问网站内的静态资源：
```
<!-- 静态资源(js、image等)的访问 -->
<mvc:default-servlet-handler/>
```

再开启 `SpringMVC` 注解模式，由于我们利用注解方法来进行相关定义，可以省去很多的配置：
```
<!-- 开启注解 -->
<mvc:annotation-driven/>
```

再进行视图解析器的相关配置：
```
    <!--ViewResolver 视图解析器-->
    <!--用于支持Servlet、JSP视图解析-->
    <bean id="jspViewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
        <property name="prefix" value="/WEB-INF/pages/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
```

关于` Controller` 如何找到视图文件，这里需要详细的说明。在 `Controller` 的一个方法中，返回的字符串定义了所需访问的 jsp 的名字（如 index）。在`jspViewResolver` 中，有两个属性，一个是 `prefix`，定义了所需访问的文件路径前缀，另一是 `suffix`，表示要访问的文件的后缀，这里为 .jsp。那么，如果返回字符串是 xxx ，SpringMVC 就会找到 /WEB-INF/pages/xxx.jsp 文件。

完成以上配置后，`dispatcher-servlet.xml` 文件如下图所示：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

    <!--指明 controller 所在包，并扫描其中的注解-->
    <context:component-scan base-package="com.hanpfei"/>

    <!-- 静态资源(js、image等)的访问 -->
    <mvc:default-servlet-handler/>

    <!-- 开启注解 -->
    <mvc:annotation-driven/>

    <!--ViewResolver 视图解析器-->
    <!--用于支持Servlet、JSP视图解析-->
    <bean id="jspViewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
        <property name="prefix" value="/WEB-INF/pages/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```

此时，我们再次尝试在 Tomcat 中运行我们的 Web 应用。从 IntelliJ 的控制台输出中可以看到如下的报错信息：
```
28-Feb-2017 16:09:42.078 INFO [RMI TCP Connection(2)-127.0.0.1] org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions Loading XML bean definitions from ServletContext resource [/WEB-INF/dispatcher-servlet.xml]
28-Feb-2017 16:09:42.129 SEVERE [RMI TCP Connection(2)-127.0.0.1] org.springframework.web.servlet.DispatcherServlet.initServletBean Context initialization failed
 org.springframework.beans.factory.xml.XmlBeanDefinitionStoreException: Line 15 in XML document from ServletContext resource [/WEB-INF/dispatcher-servlet.xml] is invalid; nested exception is org.xml.sax.SAXParseException; lineNumber: 15; columnNumber: 35; cvc-complex-type.2.4.c: 通配符的匹配很全面, 但无法找到元素 'mvc:default-servlet-handler' 的声明。
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.doLoadBeanDefinitions(XmlBeanDefinitionReader.java:399)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:336)
	at org.springframework.beans.factory.xml.XmlBeanDefinitionReader.loadBeanDefinitions(XmlBeanDefinitionReader.java:304)
	at org.springframework.beans.factory.support.AbstractBeanDefinitionReader.loadBeanDefinitions(AbstractBeanDefinitionReader.java:181)
	at org.springframework.beans.factory.support.AbstractBeanDefinitionReader.loadBeanDefinitions(AbstractBeanDefinitionReader.java:217)
	at org.springframework.beans.factory.support.AbstractBeanDefinitionReader.loadBeanDefinitions(AbstractBeanDefinitionReader.java:188)
```

提示我们的 `dispatcher-servlet.xml` 配置文件无效。在网上一通搜，找到解决方法，[关于springMVC转换json出现的异常](http://www.itdadao.com/articles/c15a612985p0.html)，修改  `dispatcher-servlet.xml` ，最终如下：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/mvc
       http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <!--指明 controller 所在包，并扫描其中的注解-->
    <context:component-scan base-package="com.hanpfei" />

    <!-- 静态资源(js、image等)的访问 -->
    <mvc:default-servlet-handler/>

    <!-- 开启注解 -->
    <mvc:annotation-driven/>

    <!--ViewResolver 视图解析器-->
    <!--用于支持Servlet、JSP视图解析-->
    <bean id="jspViewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
        <property name="prefix" value="/WEB-INF/pages/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```

然而我们再次在 Tomcat 中运行我们的 Web 应用时，再一次出错了。由控制台输出，可以看到如下的错误信息：
```
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.apache.tomcat.jdbc.pool.DataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Cannot determine embedded database driver class for database type NONE. If you want an embedded database please put a supported one on the classpath. If you have database settings to be loaded from a particular profile you may need to active it (no profiles are currently active).
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:189)
	at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:588)
	... 66 more
Caused by: org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Cannot determine embedded database driver class for database type NONE. If you want an embedded database please put a supported one on the classpath. If you have database settings to be loaded from a particular profile you may need to active it (no profiles are currently active).
	at org.springframework.boot.autoconfigure.jdbc.DataSourceProperties.determineDriverClassName(DataSourceProperties.java:229)
	at org.springframework.boot.autoconfigure.jdbc.DataSourceProperties.initializeDataSourceBuilder(DataSourceProperties.java:174)
	at org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration.createDataSource(DataSourceConfiguration.java:42)
	at org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration$Tomcat.dataSource(DataSourceConfiguration.java:53)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:162)
	... 67 more
```
看上去主要与 JDBC 配置不当有关。而在我们的小应用中，完全没有用到 JDBC，因而我们将 `spring-jdbc` 从我们的 `HelloHanpfeiSpring/web/WEB-INF/lib` 目录下删除。

再次在 Tomcat 中运行我们的 Web 应用，终于OK了。可以像前面那样，在浏览器的地址栏中输入如下内容：
```
http://localhost:8080/greeting?name=hanpfei
```
来测试我们的 Web 服务了。

# Maven 改造
为了方便工程的构建管理，我们将上面创建的工程转化为一个 Maven工程。

我们首先将 `HelloHanpfeiSpring/web/WEB-INF/lib` 目录整个删除，因为通过 Maven，我们无需在我们的工程中再放置任何 Jar 包，我们完全可以通过 Maven 的配置文件，来管理我们依赖的组件。

然后按照 Maven 的工程结构惯例调整我们工程的工程结构，最终结果将如下面这样：
![](https://www.wolfcstech.com/images/1315506-95b1cfb0de20a2c6.png)

Web应用的配置文件都被放入了 `HelloHanpfeiSpring/src/main/` 下面，`Java` 代码则被放入了 `HelloHanpfeiSpring/src/main/java` 下。

同时为了减少冲突，而直接将文件 `HelloHanpfeiSpring/src/main/com/hanpfei/Application.java` 移除。 

然后在工程的根目录添加构建、依赖配置文件 pom.xml，文件内容如下：
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>HelloHanpfeiSpring</groupId>
  <artifactId>HelloHanpfeiSpring</artifactId>

  <packaging>war</packaging>

  <version>1.0-SNAPSHOT</version>
  <name>HelloHanpfeiSpring Maven Webapp</name>
  <url>http://maven.apache.org</url>

  <properties>
    <spring.version>4.2.6.RELEASE</spring.version>
    <hibernate.version>5.1.0.Final</hibernate.version>
    <jackson.version>2.5.4</jackson.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>${spring.version}</version>
    </dependency>

    <dependency>
      <groupId>org.springframework.data</groupId>
      <artifactId>spring-data-jpa</artifactId>
      <version>1.10.1.RELEASE</version>
    </dependency>

    <dependency>
      <groupId>org.hibernate</groupId>
      <artifactId>hibernate-entitymanager</artifactId>
      <version>${hibernate.version}</version>
    </dependency>

    <dependency>
      <groupId>org.hibernate</groupId>
      <artifactId>hibernate-c3p0</artifactId>
      <version>${hibernate.version}</version>
    </dependency>

    <dependency>
      <groupId>com.mchange</groupId>
      <artifactId>c3p0</artifactId>
      <version>0.9.5.2</version>
    </dependency>

    <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>jstl</artifactId>
      <version>1.2</version>
    </dependency>

    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.39</version>
    </dependency>

    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-core</artifactId>
      <version>${jackson.version}</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>${jackson.version}</version>
    </dependency>
  </dependencies>

  <build>
    <finalName>HelloHanpfeiSpring</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

然后选择工作窗口最右侧的 `Maven Projects`，并点击弹出来的窗口左上角的刷新按钮，等待 Maven 帮我们刷新工程，下载依赖的所有包。最终整个工程的结构将如下面这样：
![](https://www.wolfcstech.com/images/1315506-7bacab8fb0b86b33.png)

我们可以通过最右侧 `Plugins` 下的 `war` 下的 `war:war` 来为我们的 Web 应用构建 war 包。构建之后，`.war` 文件将位于 `HelloHanpfeiSpring/target/` 目录下。

通过一个非常简单的 RESTful Web服务的构建，来了解在 IntelliJ + Tomcat + Maven + SpringMVC 环境下，开发 Web 服务的过程，至此结束。

#### [打赏](https://www.wolfcstech.com/about/donate.html)

# 参考文档：
[Building a RESTful Web Service](http://spring.io/guides/gs/rest-service/#initial)

[使用IntelliJ IDEA开发SpringMVC网站（一）开发环境](https://my.oschina.net/gaussik/blog/385697)

[使用IntelliJ IDEA开发SpringMVC网站（二）框架配置](https://my.oschina.net/gaussik/blog/513353)