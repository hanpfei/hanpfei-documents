----
title: Jenkins 在 Tomcat 中的部署及代码静态检查工具集成
date: 2018-04-03 21:17:49
categories: 开发流程
tags:
- 开发流程
----

# Jenkins 的简单部署

在安装了 Jenkins 运行所需的依赖（主要是 JDK）之后，可以通过如下步骤简单快速地部署 Jenkins：
<!--more-->
1. [下载 Jenkins](http://mirrors.jenkins.io/war-stable/latest/jenkins.war).

2. 打开终端并切换至下载目录。

3.  运行命令 `java -jar jenkins.war --httpPort=8080`。
`--httpPort` 参数用于指定 Jenkins 服务运行的端口。这条命令将运行 Jenkins 服务。

4.  打开浏览器并输入网址 `http://localhost:8080`。
URL 中的端口需要与上面运行 Jenkins 时指定的端口一致。在浏览器中我们能看到 Jenkins 的页面了。

5.  按照指示完成安装过程。
安装插件，并对 Jenkins 做配置。

# Jenkins 在 Tomcat 中的部署

虽然上面的 Jenkins 部署很方便快捷，但是服务管理却不是很方便。Jenkins 作为一个 Java Web 应用，其 war 包可以非常方便的部署在 Tomcat 容器中。Tomcat 在 Ubuntu 系统中的安装过程可以参考 [Ubuntu 16.04 Tomcat 8安装指南](https://www.wolfcstech.com/2017/02/24/Install_Tomcat8_on_Ubuntu16.04/) 一文。

要在 Tomcat 容器中部署 Jenkins，只需将 jenkins.war 复制到 `$TOMCAT_HOME/webapps`，然后通过 URL  http://yourhost/jenkins 访问即可。

如果 Tomcat 容器中只部署 Jenkins 服务，可以移除 `$TOMCAT_HOME/webapps` 目录中的所有内容，然后将 jenkins.war 放进这个目录中并重命名为 ROOT.war（注意大小写要保持一致）。Tomcat 将展开这个文件并创建 ROOT 目录，然后我们应该可以在 http://yourhost 看到 Jenkins，而无需任何额外的路径（如果采用了 Tomcat 的默认配置，应该是 http://yourhost:8080）

如 [Ubuntu 16.04 Tomcat 8安装指南](https://www.wolfcstech.com/2017/02/24/Install_Tomcat8_on_Ubuntu16.04/) 一文的介绍，如果为 Tomcat 建立了 systemd 服务文件，可以通过如下命令重启 Tomcat 服务：
```
$ sudo systemctl stop tomcat
$ sudo systemctl start tomcat
```

并可以通过如下命令查看 Tomcat 的运行状态：
```
$ sudo systemctl status tomcat
```

在 Jenkins 服务启动之前，设置环境变量 `JENKINS_HOME` 可以指定 Jenkins 服务的主目录。这可以通过为 Tomcat 的 systemd 服务文件 `/etc/systemd/system/tomcat.service` 添加如下的行来实现：
```
Environment=JENKINS_HOME=/opt/tomcat/jenkins_home
```

这行配置在启动 Tomcat 容器之前，导出环境变量。

更多关于在 Tomcat 中部署 Jenkins 的内容可以参考 [*Tomcat* - *Jenkins* - *Jenkins* *Wiki*](https://wiki.jenkins.io/display/JENKINS/Tomcat)。

# nginx 反向代理

将 Tomcat 容器作为 Web 服务器不是那么的方便，通过 nginx 做反向代理会好很多。在 Ubuntu 中可以通过如下命令安装并运行 nginx：
```
$ sudo apt-get install nginx
$ sudo nginx
```

修改 nginx 的配置文件 `/etc/nginx/nginx.conf`，在 `http` 块的最后添加如下几行为 Jenkins 设置反向代理：
```
        server {
            listen 80;
            server_name 59.111.103.32;
            location / {
               proxy_pass http://127.0.0.1:8080;
                proxy_redirect off;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
```

随后，就可以通过 http://yourhost URL，经过 nginx 访问 Jenkins 服务了。登陆之后，打开 Jenkins 左上角的 ***Jenkins*** -> ***Manage Jenkins*** -> ***Configure System*** Jenkins 全局配置页面，滚动到 ***Jenkins Location*** 的部分，正确配置 ***Jenkins URL***：

![](https://www.wolfcstech.com/images/1315506-445e10a5742089cb.jpg)

Jenkins 作为一个强大的持续集成平台，其强大之处的重要体现就是，支持许许多多的插件，可以将功能强大的第三方工具集成进来，代码质量保障相关的工具，比如代码的静态检查工具，是其中比较常用的一些。常用的代码静态检查工具有 PMD，FindBugs，Android Lint，CheckStyle 和 SonarQube Scanner 等。

# PMD

PMD 是一个可扩展跨语言的静态代码分析器。它查找常见的编程缺陷，如未使用的变量，空 catch 块，不必要的对象创建，等等。此外它还包含 CPD，复制粘贴探测器。CPD 查找重复代码。

PMD 扫描 ***Java 和其它编程语言*** 的源代码，并查找像下面这样的潜在问题：

 * 可能的 bugs - 空的 try/catch/finally/switch 声明
 * 死码 - 未使用的本地变量，参数和私有方法
 * 次优代码 - 无用的 String/StringBuffer 使用
 * 过于复杂的表达式 - 不必要的 if 声明，可能可以写成 while 的 for 循环

CPD，复制粘贴探测器，查找多种语言的重复代码：

 * 重复代码常常是由复制粘贴产生的。这意味着，bugs 也被复制粘贴了。修正它们意味着，修正所有重复的代码。

在正式开始集成 PMD 之前，首先需要通过 Jenkins 左上角的 ***Jenkins*** -> ***Manage Jenkins*** -> ***Manage Plugins***，在 Jenkins 中安装 PMD 的插件：

![](https://www.wolfcstech.com/images/1315506-c98b2e793e1bec91.png)

我们通过为 Jenkins Project 添加 Post-build Action 来集成 PMD。打开 Jenkins  Project 的主页，点击左边的 `Configure` 打开工程的配置页面，找到页面最下边的 `Post-build Actions`，点击 `Add post-build action` 的下拉箭头，并找到 `Publish PMD analysis results`，添加 PMD post-build action，如下图：

![](https://www.wolfcstech.com/images/1315506-0376c9b201833b8f.png)

在 `PMD results` 输入框中输入 PMD 检查结果文件的路径，这个结果文件需要我们在构建期间调用 PMD 工具生成。配置了 PMD post-build action 之后，点击左下角的 `Save` 按钮保存退出配置。

使用 PMD 工具生成源代码的静态检查分析报告的方法如下：
```
$ cd $HOME
$ wget https://github.com/pmd/pmd/releases/download/pmd_releases%2F6.2.0/pmd-bin-6.2.0.zip
$ unzip pmd-bin-6.2.0.zip
$ alias pmd="$HOME/pmd-bin-6.2.0/bin/run.sh pmd"
$ pmd -d /usr/src -f xml -r pmd.xml -rulesets java-basic,java-design
```

PMD 工具的 `-d` 参数用于指定项目的源码路径，`-f` 参数用于指定输出报告文件的格式，`-r` 用于指定输出报告文件的文件名，`-rulesets` 则用于指定检查规则集合。这个命令将产生名为 `pmd.xml` 的 XML 格式的检查报告，这也是 Jenkins 的 PMD 插件所支持的格式。

在下载并安装 PMD 工具之后，在 Jenkins 工程的构建脚本中执行 PMD 工具产生检查报告，如将 PMD 检查的功能集成进一个用 Python 写的构建脚本：
```
def run_pmd(wrapper_module_name, target_module_name):
    if "PMD_ROOT_PATH" not in os.environ.keys():
        print("Cannot find PMD")
        return

    pmd_root_path = os.environ["PMD_ROOT_PATH"]
    pmd_cmd_path = pmd_root_path + os.path.sep + "bin/run.sh"
    src_dir = wrapper_module_name + os.path.sep + target_module_name + os.path.sep + "src/main/java"
    pmd_cmd_path = "%s pmd -d %s -f xml -r pmd.xml -rulesets java-basic,java-design" % (pmd_cmd_path, src_dir)

    print(pmd_cmd_path)

    os.system(pmd_cmd_path)
```

Jenkins 工程在构建结束之后，将根据配置的 `PMD results` 路径查找 PMD 检查的结果，并将结果展示在 Jenkins 工程的主页面上，如下图所示：

![](https://www.wolfcstech.com/images/1315506-2d0b3e8a817bb7e7.png)

点击 `PMD Trend` 将可以看到 PMD 检查的详细结果，如下图：

![](https://www.wolfcstech.com/images/1315506-8c1b42e146acb8d2.png)

关于 PMD 工具用法更详细的内容，可以参考它的 [主页](https://pmd.github.io/pmd-6.2.0/) 和 [官方文档](https://pmd.github.io/pmd-6.2.0/)。

# FindBugs

FindBugs 是另一个强大的静态代码检查工具，它主要用于查找 ***Java 代码*** 中的 bugs，它查找 正确性 bugs，糟糕的做法及 Dodgy 等[问题](http://findbugs.sourceforge.net/demo.html)。

与在 Jenkins 中集成 PMD 类似，同样需要先在 Jenkins 中为 FindBugs 安装插件：

![](https://www.wolfcstech.com/images/1315506-e4769bf6fde33ca5.png)

然后配置 Jenkins 工程，添加 post-build action `Publish FindBugs analysis results`，如下图：

![](https://www.wolfcstech.com/images/1315506-0ee0700fe76db0ef.png)

`FindBugs results` 输入框中需要输入 FindBugs 工具代码检查的结果文件。Jenkins 将在构建结束之后，扫描这个文件，并在页面中展示出来。

在 Jenkins 工程的构建阶段，需要调用 FindBugs 工具生成检查报告，方法如下：
```
$ cd $HOME
$ wget https://jaist.dl.sourceforge.net/project/findbugs/findbugs/3.0.1/findbugs-3.0.1.tar.gz
$ tar xvf findbugs-3.0.1.tar.gz
$ findbugs-3.0.1/bin/findbugs -textui -low -xml -output findbugs.xml module/build/intermediates/bundles/debug/classes.jar
```

FindBugs 提供了两种用户界面，分别是 GUI 和命令行用户界面，在 Jenkins 的构建脚本中，我们以命令行界面执行 `findbugs`，这通过 `-textui` 参数来指定。`-low` 参数用于指明希望输出所有类型的问题，`-xml` 参数用于指定生成的检查报告的文件格式，`-output` 参数指明输出文件名，最后是模块编译生成的 class jar 文件。

在 Jenkins 服务器上下载安装了 FindBugs 工具之后，集成代码检查过程进 Python 构建脚本的方法大体如下：
```
def run_findbugs(wrapper_module_name, target_module_name):
    if "FIND_BUGS_ROOT_PATH" not in os.environ.keys():
        print("Cannot find findbugs")
        return
    findbugs_root_path = os.environ["FIND_BUGS_ROOT_PATH"]
    findbugs_cmd_path = findbugs_root_path + os.path.sep + "bin/findbugs"

    class_file_path = wrapper_module_name + os.path.sep + target_module_name + os.path.sep + "build/intermediates/bundles/debug/classes.jar"
    findbugs_cmd_path = "%s -textui -low -xml -output findbugs.xml %s" % (findbugs_cmd_path, class_file_path)

    print(findbugs_cmd_path)

    os.system(findbugs_cmd_path)
```

在 Jenkins 的构建任务结束之后，它扫描 FindBugs 的检查结果，并展示出来：

![截图_2018-04-03_11-00-44.png](https://www.wolfcstech.com/images/1315506-29758e2e07b59a6c.png)

点进去可以查看更详细的找到的问题：

![](https://www.wolfcstech.com/images/1315506-eb0216367b88b45e.png)

关于 FindBugs 更详细的内容，可以参考其[主页](http://findbugs.sourceforge.net/)和[文档](http://findbugs.sourceforge.net/manual/index.html)。

总结一下 PMD、FindBugs 集成进 Jenkins 的流程：

 * 全局 Jenkins 配置方面，为相应的代码静态检查工具安装插件。
 * 在 Jenkins 工程配置中，为相应的代码静态检查工具添加 post-build action，配置检查结果文件的存放路径。
 * 为 Jenkins 服务器下载并安装代码静态检查工具。
 * 在 Jenkins 工程的构建脚本中，调用代码检查工具生成检查报告文件。

其它的代码静态检查工具集成进 Jenkins 的过程与此类似，如 Checkstyle 和 Android Lint。

# Checkstyle

Checkstyle 是一个帮助程序员编写符合某一编码规范的 ***Java 代码*** 的开发工具。为它提供编码规范的定义文件和源代码，它自动检查源代码中不符合规范的地方。编码规范的定义文件可以自行配置，比较常用的 Java 代码编码规范如 [Sun 代码规范](http://www.oracle.com/technetwork/java/javase/documentation/codeconvtoc-136057.html) 和 [Google Java 代码规范](http://checkstyle.sourceforge.net/reports/google-java-style-20170228.html)。

在 Jenkins 中集成 Checkstyle 的整体过程与集成 PMD 和 FindBugs 的过程类似。需要为 Checkstyle 安装的插件为 `Checkstyle Plug-in`，需要为 Jenkins 工程添加的 post-build action 为 `Publish Checkstyle analysis results`。下载并安装 Checkstyle 工具的方法如下：
```
$ wget https://excellmedia.dl.sourceforge.net/project/checkstyle/checkstyle/8.8/checkstyle-8.8-bin.tar.gz
$ tar xvf checkstyle-8.8-bin.tar.gz
```

此外还需要自己定义或下载公开的代码风格定义文件，如 Sun 代码规范 [sun_checks.xml](https://raw.githubusercontent.com/checkstyle/checkstyle/master/src/main/resources/sun_checks.xml) 和 Google Java 代码规范 [google_checks.xml](https://raw.githubusercontent.com/checkstyle/checkstyle/master/src/main/resources/google_checks.xml)

执行 Checkstyle 工具生成检查报告的方法如下：
```
$ java -jar checkstyle-8.8/checkstyle-8.8-all.jar com.puppycrawl.tools.checkstyle.Main -c checkstyle_config/google_checks.xml module/src/main/java -f xml -o checkstyle-result.xml
```

Checkstyle 工具的 `-c` 参数用于指定代码风格的定义文件，`-f` 参数用于指定用于指定输出检查报告文加的格式，`-o` 参数用于指定输出报告文件的文件名，同时需要为 Checkstyle 指定项目的 Java 源代码路径。上面的命令中 Checkstyle 将输出文件名为 `checkstyle-result.xml` 格式为 xml 的检查报告。

将 Checkstyle 工具集成进 Python 的工程构建脚本的方法如下：
```
def run_checkstyle(wrapper_module_name, target_module_name):
    if "CHECKSTYLE_ROOT_PATH" not in os.environ.keys():
        print("Cannot find Checkstyle")
        return
    jar_path = os.environ["CHECKSTYLE_ROOT_PATH"] + os.path.sep + "checkstyle-8.8-all.jar"
    checkstyle_cmd = "java -jar " + jar_path + " com.puppycrawl.tools.checkstyle.Main"
    script_path = __file__
    config_file_path = os.path.dirname(script_path) + os.path.sep + "checkstyle_config/google_checks.xml"
    checkstyle_cmd = checkstyle_cmd + " -c " + config_file_path
    src_dir = " " + wrapper_module_name + os.path.sep + target_module_name + os.path.sep + "src/main/java"
    checkstyle_cmd = checkstyle_cmd + src_dir + " -f xml -o checkstyle-result.xml"

    print(checkstyle_cmd)

    os.system(checkstyle_cmd)
```

Jenkins 在工程构建结束之后，扫描 Checkstyle 的检查报告，并展示出来：

![截图_2018-04-03_11-34-05.png](https://www.wolfcstech.com/images/1315506-55b435b08191794b.png)

Checkstyle 更详细的用法可以参考其[主页](http://checkstyle.sourceforge.net/) 和它的 [命令行用法说明文档](http://checkstyle.sourceforge.net/cmdline.html)。

# Android Lint
将 Android Lint 集成进 Jenkins 的过程，与前面的那些 PMD、FindBugs 和 Checkstyle 的过程类似，只是需要安装的 Jenkins 插件为 [Android Lint Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Android+Lint+Plugin)，需要为 Jenkins 工程添加的 post-build action 为 `Publish Android Lint results`。

更为简单的是，Android Lint 是 Android Sdk 中的工具，因而无需单独下载安装。对于 Gradle 工程而言，甚至无需单独运行 Android lint 工具，而只需运行 `lintDebug` 或 `lintRelease` gradle 任务即可，它们将在模块的 `build/reports/lint-results*.xml` 处生成 检查报告。

在 Python 的 Jenkins 工程构建脚本中生成 Android Lint 检查报告的方法如下：
```
def run_android_lint(target_module_name):
    lint_task_name = ":%s:lintDebug" % target_module_name
    os.system("./gradlew " + lint_task_name)
```

Android Lint 的检查结果类似于下面这样：

![](https://www.wolfcstech.com/images/1315506-916733363733cf6f.png)

# SonaQube Scanner

SonaQube 是一个开源的代码质量分析管理平台，它专注于持续地分析和测量技术方面的质量，从项目组合到方法。通过插件，它可以支持包括 Java，C#，C/C++，PL/SQL，Cobol，JavaScrip，Groovy 等在内的 [20 多种编程语言](https://www.sonarsource.com/products/codeanalyzers/) 的代码质量检测与管理。

SonaQube 对代码质量的管理通过 Web 服务 SonaQube 提供，代码质量检测通过 SonaQube Scanner 完成。通常的流程是，SonaQube Scanner 执行代码的静态检查分析，然后将检查的结果传给 SonaQube 服务，SonaQube 服务解析并展示分析结果。

SonaQube Scanner 可以集成进 MSBuild，Maven，Gradle，Ant 等构建系统中，当然也可以集成进 Jenkins 或在命令行上运行。

## 安装 SonaQube 服务
在 Jenkins 中集成 SonaQube Scanner，分析结果依然需要由 SonaQube 服务来解析与展示，因而首先需要安装 SonaQube 服务。在 SonarQube 的[下载页面](https://www.sonarqube.org/downloads/) 选择需要的版本并下载，如 SonarQube 6.7.2 (LTS *) 。

下载完成后，执行如下命令安装并启动 SonaQube 服务：
```
$ unzip sonarqube-6.7.2.zip
$ sonarqube-6.7.2/bin/linux-x86-64/sonar.sh start
```

SonarQube 自带数据库和 Web 服务器，因而通过上面简单的两条命令，就可以将 SonaQube 服务运行起来了。默认情况下，SonaQube 服务运行于 9000 端口上：

![](https://www.wolfcstech.com/images/1315506-491c8b952e81fad0.png)

但需要注意不能以 root 用户启动 SonaQube 服务，否则将启动失败。通过 `sonarqube-6.7.2/logs/` 目录下的 ElasticSearch 日志文件 es.log 和 SonaQube 日志文件 sonar.log 可以看到这一点。为了获得更好的性能和稳定性，可以使用外部的数据库服务， SonaQube 服务对此提供了良好的支持。关于 SonaQube 服务安装配置更详细的过程，可以参考 SonaQube 的官方文档 [Installing the Server](https://docs.sonarqube.org/display/SONAR/Installing+the+Server)。

## 命令行运行 SonaQube Scanner

SonaQube Scanner 可以集成进 MSBuild，Maven，Gradle，Ant 及 Jenkins 等工具中，也可以在命令上独立运行。首先下载并安装 SonaQube Scanner：
```
$ wget https://sonarsource.bintray.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-3.1.0.1141-linux.zip
$ unzip sonar-scanner-cli-3.1.0.1141-linux.zip
```

然后为要检查的工程编写属性配置文件 sonar-project.properties，如：
```
java-module.sonar.projectName=Java Module
java-module.sonar.language=java
# .表示projectBaseDir指定的目录
java-module.sonar.sources=study-project/lib-module-question/src/main/java
java-module.sonar.projectBaseDir=study-project/lib-module-question
sonar.binaries=classes
sonar.projectKey=study-project
sonar.sources=study-project/lib-module-question/src/main/java
sonar.java.binaries=study-project/lib-module-question/build/intermediates/bundles/debug/classes.jar
```

上面的配置中，`sonar.projectKey` 用于指定工程名称，工程名称标识一个工程，SonaQube Scanner 在检查结束之后向 SonaQube 服务发布检查结果，SonaQube 服务以工程为单位展示检查结果。`sonar.sources` 用于指定要检查的源码的路径。`sonar.java.binaries` 用于指定编译生成的 jar 文件的路径。

在命令行中，在 sonar-project.properties 文件的相同目录下，执行如下命令：
```
$ sonar-scanner-3.1.0.1141-linux/bin/sonar-scanner
```

它将根据 sonar-project.properties 配置文件的内容静态分析源码，并将结果发布给 SonaQube 服务。默认情况下，分析结果将发布给位于 `http://localhost:9000` 的 SonaQube 服务。但可以通过如下配置项来进行定制：
```
sonar.host.url=http://localhost:9000
sonar.login=apitoken/username
sonar.password=blank/passwd
```

关于 SonaQube Scanner 执行、配置更详细的信息，可以参考官方文档 [Analyzing with SonarQube Scanner](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner)、[Advanced SonarQube Scanner Usages](https://docs.sonarqube.org/display/SCAN/Advanced+SonarQube+Scanner+Usages)
 和 [Analysis Parameters](https://docs.sonarqube.org/display/SONAR/Analysis+Parameters)。

## 在 Jenkins 中集成 SonaQube Scanner

同样需要先为 SonaQube Scanner 安装插件，这次为 `SonarQube Scanner for Jenkins`。然后以管理员身份登录 Jenkins，并进入 ***Manage Jenkins*** > ***Configure System***。向下滚动到 ***SonarQube configuration*** 的部分，点击 ***Add SonarQube***，然后按提示输入对应的值，如下图这样：

![](https://www.wolfcstech.com/images/1315506-5c1a68eeacad2f58.png)

其中 `Name` 一栏输入 SonaQube 的名称，名称任意取。`Server URL` 一栏输入 SonaQube 服务的 Url。`Server authentication token` 输入用户认证 token，这个 token 是由 SonaQube 服务生成的。

登录 SonaQube 服务（如第一次以 admin（用户名）/admin（密码）登录 SonaQube 服务）之后，点击右上角的用户图标，选择 ***My Account***，打开账户主页，并选择 ***Security***：

![](https://www.wolfcstech.com/images/1315506-61712adb0ee66fb6.png)

在 ***Generate New Token:*** 输入框中输入 token 名称，token 名称可任意选择，然后点击 ***Generate*** 生成 token。如 SonaQube 服务给出的提示，生成的 token 需要复制出去，这个 token 将无法被再次看到。这个 token 即是前面需要提供给 Jenkins 的 `Server authentication token`。

然后执行 Jenkins 全局的一些配置，具体而言，是添加 SonarQube Scanner：

1. 以管理员身份登录 Jenkins，然后进入 ***Manage Jenkins*** -> ***Global Tool Configuration***。
2. 滚动到 ***SonarQube Scanner*** 配置部分，并点击 ***Add SonarQube Scanner***。它基于典型的 Jenkins 工具自动安装。当然也可以选择指向一个已经安装的 SonarQube Scanner 版本（反选 'Install automatically'）。

最后是 Jenkins 工程的配置：

1. 打开 Jenkins 工程的配置页面，滚动到 ***Build*** 部分。
2. 为 Jenkins 工程的构建添加 *SonarQube Scanner* 构建步骤。
3. 配置 SonarQube 分析属性。也可以指向一个已有的 *sonar-project.properties* 文件或直接在 ***Analysis properties*** 字段设置分析属性，如下图：

![](https://www.wolfcstech.com/images/1315506-99cf50f73e0855e8.png)

然后点击左下角的 `Save`，保存并结束配置。后面在执行 Jenkins Project 的构建任务时，SonarQube Scanner 将执行，生成分析报告并发送给 SonaQube 服务，报告有 SonaQube 解析并展示。更多内容请参考 [Analyzing with SonarQube Scanner for Jenkins](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Jenkins)。

参考文档

[Getting started with the Guided Tour](https://jenkins.io/doc/pipeline/tour/getting-started/)
[Tomcat - Jenkins - Jenkins Wiki](https://wiki.jenkins.io/display/JENKINS/Tomcat)
[Analyzing with SonarQube Scanner for Jenkins](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Jenkins)
[使用 Jenkins 与 Sonar 集成对代码进行持续检测](https://www.ibm.com/developerworks/cn/devops/1612_qusm_jenkins/index.html)
[SonarQube代码质量管理平台安装与使用](https://blog.csdn.net/hunterno4/article/details/11687269)
[Analyzing Source Code](https://docs.sonarqube.org/display/SCAN/Analyzing+Source+Code)

**除非其它特别说明，你可以转载这篇文章，但请加入文章作者和出处。谢谢。**

Dong.
