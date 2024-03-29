---
title: Gerrit代码审核服务器搭建全过程
date: 2017-11-24 19:05:49
categories: 开发流程
tags:
- 开发流程
- Java开发
---

# 建立专有帐户
```
$ sudo useradd gerrit -m -s /bin/bash
$ sudo passwd gerrit
$ su gerrit
```
<!--more-->
# 配置 Java 环境

# 从官网下载gerrit

https://www.gerritcodereview.com/

当前最新版本为 2.14。

# 安装 MySQL
```
$ sudo apt-get install mysql-server
$ sudo apt-get install mysql-client
$ sudo apt-get install libmysqlclient-dev
```

# 安装gerrit

通过如下命令安装 Gerrit：
```
$ java -jar gerrit-2.14.war init -d review_site
```

按照提示一步步完成安装。

有几个按转配置需要特别注意一下。

关于 Gerrit 的 Git 仓库的保存地址：
```
*** Git Repositories
*** 

Location of Git repositories   [git]: GerritResource
```

这个选项用于配置 Gerrit 的 Git 仓库的保存地址。上面的配置将创建 `/home/gerrit/review_site/GerritResource` 目录用于保存 Gerrit 的 Git 仓库。

关于数据库的配置：
```
*** SQL Database
*** 

Database server type           [h2]: mysql

Gerrit Code Review is not shipped with MySQL Connector/J 5.1.41
**  This library is required for your configuration. **
Download and install it now [Y/n]? Y
Downloading https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.41/mysql-connector-java-5.1.41.jar ... OK
Checksum mysql-connector-java-5.1.41.jar OK
Server hostname                [localhost]: cloudgame-codereview.com
Server port                    [(mysql default)]: 
Database name                  [reviewdb]: 
Database username              [gerrit]: 
gerrit's password              : 
              confirm password :
```

这里选择 MySQL 作为 Gerrit 的数据库，其它选项全部采用默认配置。对于这种选择，需要连上 MySQL，为 Gerrit 创建相应的数据库，用户，并为用户授权：
```
$ mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 35
Server version: 5.7.20-0ubuntu0.16.04.1 (Ubuntu)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SELECT USER();
+----------------+
| USER()         |
+----------------+
| root@localhost |
+----------------+
1 row in set (0.00 sec)

mysql> create database reviewdb;
Query OK, 1 row affected (0.01 sec)

mysql> CREATE USER 'gerrit'@'localhost' IDENTIFIED BY 'gerrit';
Query OK, 0 rows affected (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> grant all privileges on reviewdb.* to gerrit@localhost identified by 'gerrit';
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| reviewdb           |
| sys                |
+--------------------+
8 rows in set (0.01 sec)

mysql>
```

Gerrit 安装过程中，可能会报出如下的 Exception：
```
Exception in thread "main" com.google.gwtorm.server.OrmException: Cannot apply SQL
CREATE TABLE account_group_members_audit (
added_by INT DEFAULT 0 NOT NULL,
removed_by INT,
removed_on TIMESTAMP NULL DEFAULT NULL,
account_id INT DEFAULT 0 NOT NULL,
group_id INT DEFAULT 0 NOT NULL,
added_on TIMESTAMP NOT NULL
,PRIMARY KEY(account_id,group_id,added_on)
)
	at com.google.gwtorm.jdbc.JdbcExecutor.execute(JdbcExecutor.java:44)
	at com.google.gwtorm.jdbc.JdbcSchema.createRelations(JdbcSchema.java:134)
	at com.google.gwtorm.jdbc.JdbcSchema.updateSchema(JdbcSchema.java:104)
	at com.google.gerrit.server.schema.SchemaCreator.create(SchemaCreator.java:81)
	at com.google.gerrit.server.schema.SchemaUpdater.update(SchemaUpdater.java:108)
	at com.google.gerrit.pgm.init.BaseInit$SiteRun.upgradeSchema(BaseInit.java:386)
	at com.google.gerrit.pgm.init.BaseInit.run(BaseInit.java:143)
	at com.google.gerrit.pgm.util.AbstractProgram.main(AbstractProgram.java:61)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.google.gerrit.launcher.GerritLauncher.invokeProgram(GerritLauncher.java:204)
	at com.google.gerrit.launcher.GerritLauncher.mainImpl(GerritLauncher.java:108)
	at com.google.gerrit.launcher.GerritLauncher.main(GerritLauncher.java:63)
	at Main.main(Main.java:24)
Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Invalid default value for 'added_on'
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.mysql.jdbc.Util.handleNewInstance(Util.java:425)
	at com.mysql.jdbc.Util.getInstance(Util.java:408)
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:943)
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3973)
	at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3909)
	at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2527)
	at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2680)
	at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2497)
	at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2455)
	at com.mysql.jdbc.StatementImpl.executeInternal(StatementImpl.java:839)
	at com.mysql.jdbc.StatementImpl.execute(StatementImpl.java:739)
	at com.google.gwtorm.jdbc.JdbcExecutor.execute(JdbcExecutor.java:42)
	... 15 more
```

这个异常可通过如下方式解决：使用 MySQL root 用户登录，设置 `set global explicit_defaults_for_timestamp=1; `，像下面这样：
```
$ mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 76
Server version: 5.7.20-0ubuntu0.16.04.1 (Ubuntu)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> set global explicit_defaults_for_timestamp=1;
Query OK, 0 rows affected (0.00 sec)

mysql> exit;
Bye
```

然后重新安装 Gerrit 即可。（新版本的 MySQL TIMESTAMP 默认值的问题需要手动配置一下。）

安装完成后，Gerrit Server 将自动启动。

# 配置gerrit
```
$ vim review_site/etc/gerrit.config
```

这个配置文件的内容如下：
```
[gerrit]
	basePath = GerritResource
	serverId = 7b8058ff-932a-41ed-a1fa-6ea53dfba8e1
	canonicalWebUrl = http://review.virtcloudgame.com:8080/
        useSSL=false
[database]
	type = mysql
	hostname = review.virtcloudgame.com
	database = reviewdb
	username = gerrit
[index]
	type = LUCENE
[auth]
	type = http
[receive]
	enableSignedPush = false
[sendemail]
	smtpServer = localhost
[container]
	user = gerrit
	javaHome = /usr/lib/jvm/java-8-openjdk-amd64/jre
[sshd]
	listenAddress = *:29418
[download]
        scheme = ssh
        scheme = http
[httpd]
        listenUrl = proxy-http://127.0.0.1:8080/
[cache]
        directory = cache
[plugins]
        allowRemoteAdmin = true
```

按照如上内容配置完 Gerrit Server 之后，可以通过如下命令重新启动它以应用新的配置：
```
$ review_site/bin/gerrit.sh restart
```

# 设置第一个 Gerrit 用户的帐号和密码
```
$ touch ./review_site/etc/passwd
$ htpasswd -b ./review_site/etc/passwd admin admin
Adding password for user admin
```

`htpasswd` 命令是 `apache2-utils` 软件包中的一个工具。如果系统中还没有安装的话，通过如下命令进行安装：
```
$ sudo apt-get install apache2-utils
```

(后续再添加 Gerrit 用户可使用 `htpasswd -b ./review_site/etc/passwd UserName PassWord`) 

对于 Gerrit 来说，第一个成功登录的用户具有特殊意义 —— 它会直接被赋予管理员权限。对于第一个账户，需要特别注意一下。

# 开启 Gerrit 服务器
Gerrit 服务器还可以通过如下的命令进行启动：
```
$ review_site/bin/gerrit.sh start
Starting Gerrit Code Review: FAILED
```

上面的命令，通过 `review_site/bin/gerrit.sh start` 启动 Gerrit Server，但是失败了。Gerrit Server 启动，需要监听 8080 端口，如前面的配置文件 `review_site/etc/gerrit.config` 中的 `listenUrl` 行所显示的那样。通过如下的命令查看 8080 端口的使用情况：
```
$ sudo lsof -i -P | grep 8080
java       9538          tomcat   53u  IPv6 17098680      0t0  TCP *:8080 (LISTEN)
```

可以看到 8080 端口被 tomcat 用户下的某个 Java 进程占用了。我们可以杀掉相应的应用来解决 Gerrit Server 启动失败的问题（Tomcat 的安装配置启动按照[Ubuntu 16.04 Tomcat 8安装指南](https://www.wolfcstech.com/2017/02/24/Install_Tomcat8_on_Ubuntu16.04/) 一文的说明进行）：
```
$ sudo systemctl stop tomcat
$ sudo systemctl status tomcat
● tomcat.service - Apache Tomcat Web Application Container
   Loaded: loaded (/etc/systemd/system/tomcat.service; enabled; vendor preset: enabled)
   Active: inactive (dead) since 四 2017-11-23 10:23:02 CST; 4s ago
  Process: 9805 ExecStop=/opt/tomcat/bin/shutdown.sh (code=exited, status=0/SUCCESS)
  Process: 9527 ExecStart=/opt/tomcat/bin/startup.sh (code=exited, status=0/SUCCESS)
 Main PID: 9538 (code=exited, status=0/SUCCESS)

11月 23 10:20:18 ThundeRobot startup.sh[9527]: Removing/clearing stale PID file.
11月 23 10:20:18 ThundeRobot systemd[1]: Started Apache Tomcat Web Application Container.
11月 23 10:23:01 ThundeRobot systemd[1]: Stopping Apache Tomcat Web Application Container...
11月 23 10:23:01 ThundeRobot shutdown.sh[9805]: Using CATALINA_BASE:   /opt/tomcat
11月 23 10:23:01 ThundeRobot shutdown.sh[9805]: Using CATALINA_HOME:   /opt/tomcat
11月 23 10:23:01 ThundeRobot shutdown.sh[9805]: Using CATALINA_TMPDIR: /opt/tomcat/temp
11月 23 10:23:01 ThundeRobot shutdown.sh[9805]: Using JRE_HOME:        /usr/lib/jvm/java-1.8.0-openjdk-amd64/jre
11月 23 10:23:01 ThundeRobot shutdown.sh[9805]: Using CLASSPATH:       /opt/tomcat/bin/bootstrap.jar:/opt/tomcat/bin/tomcat-juli.jar
11月 23 10:23:01 ThundeRobot shutdown.sh[9805]: Using CATALINA_PID:    /opt/tomcat/temp/tomcat.pid
11月 23 10:23:02 ThundeRobot systemd[1]: Stopped Apache Tomcat Web Application Container.
```

当然也可以通过修改 Gerrit Server 监听的端口来解决问题。

再次启动 Gerrit Server：
```
$ review_site/bin/gerrit.sh start
Starting Gerrit Code Review: OK
```

可以看到 Gerrit Server 成功启动了。此时通过浏览器，打开 `http://review.virtcloudgame.com:8080` 将可以看到如下这样的页面：

![2017-11-27 15-11-01屏幕截图.png](../images/1315506-27e86131e47160c1.png)

# 修改认证方式和反向代理
为了通过更为强大的 Web 服务器来对外提供服务，同时方便 Gerrit Server 的 HTTP 用户认证方式可以正常工作，需要设置反向代理。这里使用 nginx 作为 Web 服务器。

首先更改 Gerrit 配置，使能代理；另外，使用反向代理后就可以直接使用 nginx 的 80 端口访问了，需要把 `canonicalWebUrl` 中的 8080 去掉，Gerrit Server 监听的端口也改为 8081：
```
[gerrit]
	basePath = GerritResource
	serverId = 7b8058ff-932a-41ed-a1fa-6ea53dfba8e1
	canonicalWebUrl = http://review.virtcloudgame.com/
        useSSL=false
. . . . . .
[auth]
	type = HTTP
. . . . . .
[httpd]
        listenUrl = proxy-http://127.0.0.1:8081/
```

修改之后，重启 Gerrit Server：
```
$ review_site/bin/gerrit.sh restart
Stopping Gerrit Code Review: OK
Starting Gerrit Code Review: OK
```

上面的修改将使 Gerrit Server 监听在 8081 端口上，同上，将认证方式修改为 HTTP。

Gerrit Server 强制要求使用反向代理，通过反向代理服务器提供的 `Authorization` 等 HTTP 头来获得用户认证信息。

接着配置 nginx。修改 nginx 的配置文件 `/etc/nginx/nginx.conf`，在它的 `http` 块中加入如下内容：
```
        server {
	  listen 80;
	  server_name review.virtcloudgame.com;

	  location ^~ / {
            auth_basic "Restricted";
            auth_basic_user_file /home/gerrit/review_site/etc/passwd;
	    proxy_pass        http://127.0.0.1:8081;
	    proxy_set_header  X-Forwarded-For $remote_addr;
	    proxy_set_header  Host $host;
	  }
	}
```

`auth_basic_user_file` 行用户配置用户名和密码文件的保存路径。`proxy_pass` 行用户设置 Gerrit Server 的地址。

修改之后，让 nginx 重新加载配置文件：
```
$ sudo nginx -s reload
```

这样就可以直接通过 nginx 监听的 80 端口访问 Gerrit Server 了。

![2017-11-23 11-41-07屏幕截图.png](http://upload-images.jianshu.io/upload_images/1315506-c616dba14c5b09ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# Replication 配置

所谓的 replication，是 Gerrit 的一个插件，它可以自动地将 Gerrit Code Review 对它所管理的 Git 仓库创建的任何 changes push 到另外一个系统里。Gerrit 本身提供了两大功能：一是 Code Review；二是 Git 仓库。Replication 插件通常用于提供 changes 的镜像，或热备份。

此外，许多现有的项目可能是用另外一套系统来管理 Git 代码仓库的，比如 GitLab，或者 GitHub。需要引入 Gerrit 做 Code Review，同时对接这些已有的 Git 仓库系统时，replication 插件比较有用。

配置 replication 将代码同步到 GitLab 的方法如下。

如果通过 SSH 来从 Gerrit 同步代码到 GitLab，需要确保远程系统，也就是 GitLab 服务器的主机密钥已经在 Gerrit 用户的 `~/.ssh/known_hosts` 文件中了。

首先，如果还没有生成过 SSH key 的话，需要生成 SSH Key：
```
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/gerrit/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/gerrit/.ssh/id_rsa.
Your public key has been saved in /home/gerrit/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:idkrXgfm7oq+dNsWi9Q9ewDyZuQEYl8icA4ltq8o46U gerrit@admins-B85-HD3-A
The key's randomart image is:
+---[RSA 2048]----+
|  =oo            |
| . *+ o .        |
|  ...+ +         |
|   .  o++.       |
|    . oBSo       |
| . .  .oBo+      |
|+ ....o++o.+     |
|o.o. +.*o.. .    |
| E .+.+++  .     |
+----[SHA256]-----+
```

然后将 SSK Key 的公钥，即 `~/.ssh/id_rsa.pub` 文件的内容，添加到 GitLab 中具有权限的用户的 SSH key 列表里。

最简单的添加主机密钥的方法是，在命令行中手动地连接一次远程系统：
```
$ su -c 'ssh -p 22222 g.hz.netease.com echo' gerrit
密码： 
Permission denied (publickey).
```

上面的远程登录失败了，但不用管它，远程主机的密钥实际上已经添加进 `~/.ssh/id_rsa.pub` 文件里了。

注意，如果 GitLab 的 SSH 服务不是在标准的 22 端口上提供的，需要通过 `-p` 参数给上面执行的 `ssh` 命令指定 SSH 服务的端口号。GitLab 的 SSH 服务监听的端口号，可以从项目的 URL 中看出来 —— SSH URL 中主机名后面的冒号之后的是端口号。

还有另外一种方法，同样可以用来添加主机密钥，即通过 `ssh-keyscan` 命令，像下面这样：
```
$ ssh-keyscan -p 22222 -t rsa g.hz.netease.com >> /home/gerrit/.ssh/known_hosts
# g.hz.netease.com:22222 SSH-2.0-OpenSSH_6.0p1 Debian-4+deb7u6
```

同样需要注意，GitLab 不是通过标准 SSH 端口 22 来提供 SSH 服务的话，通过参数 `-p` 为 `ssh-keyscan` 命令指定 GitLab 服务器提供 SSH 服务的正确端口号。

接下来，创建 `$site_path/etc/replication.config` 文件。这是一个 Git
 风格的配置文件，它用来控制 replication 插件的设置。这个文件由一个或多个 `remote` 段组成，每个 `remote` 段都为一个或多个目标 URIs 提供公共配置设置。如：
```
[remote "g.hz.netease.com"]
    url = ssh://git@g.hz.netease.com:22222/cloudgame/${name}.git
    push = +refs/heads/*:refs/heads/*
    push = +refs/tags/*:refs/tags/*
    push = +refs/changes/*:refs/changes/*
    timtout = 30
    threads = 3
```


如果 `~/.ssh/config` 文件存在的话，Gerrit 将在启动的时读取并缓存它，并支持大部分 SSH 配置序啊想，比如：
```
Host g.hz.netease.com
    User git
    Port 22222
    IdentityFile ~/.ssh/id_rsa
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    PreferredAuthentications publickey
```

这个配置文件支持的全部选项如下：
 * Host
 * Hostname
 * User
 * Port
 * IdentityFile
 * PreferredAuthentications
 * StrictHostKeyChecking

需要注意的是这个文件的权限，文件的 "其它" 用户访问权限，不能可读写。


然后重新启动 Gerrit Server：
```
$ review_site/bin/gerrit.sh restart
```

还可以通过如下命令，重新加载 replication 插件来应用新的配置：
```
$ ssh -p 29418 localhost gerrit plugin reload replication
```

要在运行时手动地触发 replication，可以参考 SSH 命令 `start`：
```
ssh -p @SSH_PORT@ @SSH_HOST@ @PLUGIN@ start
```

比如：
```
$ ssh -p 29418 localhost replication start
```

replication 插件在执行过程中，产生的日志文件位于 `~/review_site/logs/replication_log`。当 replication 失败时，可以从这个文件中找到一点线索。如果 这个日志中提供的信息不足，还可以修改 replication 的代码，让它输出更多日志，编译它的代码，然后将生成的 jar 文件，替换 `~/review_site/plugins/replication.jar`，并重启 Gerrit Server。

# Gerrit 工程的创建
用前面创建的 `admin` 用户登录，它将自动获得管理员权限，可以以这个用户创建工程。登录之后，选择 `Projects` -> `Create New Project`：
![](../images/1315506-f1a3fd848fab6968.png)

在 `Project Name:` 一栏输入工程的名字，并点击 `Create Project` 按钮即可完成工程的创建。在 Gerrit 的配置文件 `review_site/etc/gerrit.config` 中，`basePath` 定义了这些工程的位置。

同时要注意，工程名字要与 GitLab 上对应的工程名字相同。

在 Gerrit 上创建了工程之后，还需要用 GitLab 上已有的代码替换 Gerrit 中的空工程：
```
$ cd ~/review_site/GerritResource/
$ rm -rf cloudgame_tools.git/
$ git clone --bare ssh://git@g.hz.netease.com:22222/cloudgame/cloudgame_tools.git
```

在工程的主页，可以找到 clone 工程的命令。直接复制命令，然后将工程 clone 到本地：
![](../images/1315506-0105a12ab738c99a.png)

像下面这样 Clone Gerrit 上的工程：
```
$ git clone ssh://admin@review.virtcloudgame.com:29418/EventServer && scp -p -P 29418 admin@review.virtcloudgame.com:hooks/commit-msg EventServer/.git/hooks/
正克隆到 'EventServer'...
remote: Counting objects: 255, done
remote: Finding sources: 100% (255/255)
remote: Total 255 (delta 156), reused 255 (delta 156)
接收对象中: 100% (255/255), 412.77 KiB | 0 bytes/s, 完成.
处理 delta 中: 100% (156/156), 完成.
检查连接... 完成。
commit-msg
```

根据需要，像使用普通的 Git 工程那样，修改代码，commit，然后通过如下命令 push 代码到 Gerrit 进行 Code Review：
```
git push 远程地址 本地分支:refs/for/远程分支
```

如：
```
$ git push origin master:refs/for/master
对象计数中: 3, 完成.
Delta compression using up to 8 threads.
压缩对象中: 100% (2/2), 完成.
写入对象中: 100% (3/3), 329 bytes | 0 bytes/s, 完成.
Total 3 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: new: 1, refs: 1, done    
remote: 
remote: New Changes:
remote:   http://review.virtcloudgame.com/12 Just for test.
remote: 
To ssh://admin@review.virtcloudgame.com:29418/EventServer
 * [new branch]      master -> refs/for/master
```

在 Gerrit 上将看到刚刚提交的代码：
![2017-11-24 19-03-56屏幕截图.png](../images/1315506-74cf614902a345dc.png)

Code Review 结束之后，Submit 代码，代码将被提交到 Gerrit 的 Git 仓库，同时被复制到 GitLab 的对应仓库中。

### [打赏](https://www.wolfcstech.com/about/donate.html)

参考文档：
[Gerrit代码审核服务器搭建全过程](http://www.gerrit.com.cn/1568.html)
[Gerrit2安装配置](http://www.gerrit.com.cn/1609.html)
[gerrit使用总结](http://www.gerrit.com.cn/category/use)
[Gerrit服务器搭建](http://www.hovercool.com/en/Gerrit%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%90%AD%E5%BB%BA)
[gerrit2安装和用Nginx设置HTTP基本验证](http://blog.sina.com.cn/s/blog_75110cc50101hgbh.html)
[Gerrit Code Review - Reverse Proxy](https://gerrit-documentation.storage.googleapis.com/Documentation/2.14.5.1/config-reverseproxy.html)
[Gerrit与Gitlab同步配置replication&其他配置](https://www.bbsmax.com/A/mo5kYQWzwR/)
[CentOS安装gitlab，gerrit，jenkins并配置ci流程](http://www.cnblogs.com/juandx/p/5372373.html)
[ssh-keyscan(1) - Linux man page](https://linux.die.net/man/1/ssh-keyscan)
[Gerrit Plugins](https://gerrit-documentation.storage.googleapis.com/Documentation/2.14.5.1/config-plugins.html#replication)
[Replication Configuration](https://review.gerrithub.io/plugins/replication/Documentation/config.md)
[Replication Configuration](https://gerrit.googlesource.com/plugins/replication/+doc/master/src/main/resources/Documentation/config.md)
[使用git push命令提交到gerrit](http://www.zhixing123.cn/android/use-git-push-to-push-to-gerrit.html)
