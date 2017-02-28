---
title: 一文学会Go语言数据库操作
date: 2017-02-26 20:17:49
categories: 后台开发
tags:
- 后台开发
- Go 语言
---

对许多 Web 应用程序而言，数据库都是其核心所在。数据库几乎可以用来存储想查询和修改的任何信息，比如用户信息、产品目录或者新闻列表等。
<!--more-->
数据库是Web编程不可或缺的主题。这里我们就来看一下如何用 Go 语言操作不同的数据库。

# database/sql 接口
Go 语言不同于 PHP，它没有官方提供任何数据库驱动，而是为开发者定义了一组标准接口。提供数据库服务的开发者，可以实现这组接口，以为特定的数据提供驱动。而数据库的使用者，也可以通过这组接口方便地访问数据库，甚至在适当的时候，以非常小的代价迁移数据库。

通常我们在以类似如下的方式导入数据库驱动模块时：
```
import	_ "github.com/Go-SQL-Driver/MySQL"
```
上面的 `import` 语句中的下划线表示，我们要导入这个模块，但不会直接使用其中的符号。但导入的时候，会执行模块的初始化代码，也就是其中定义的 `init()` 函数，在这个函数中数据库驱动会自动将其自身注册进 Go 的 `database/sql` 框架中，后面我们就可以方便地通过这个接口来访问对应得数据了。

关于这组接口的详细内容，可以参考[官方文档](http://go-database-sql.org/index.html)。

# MySQL
目前网上流行的网站架构方式是 LAMP，其中的 M 即为 MySQL。作为数据库，MySQL 以免费、开源、使用方便为优势而成为了许多Web开发的后端数据库存储引擎。

## MySQL 安装
首先我们需要安装 MySQL，这可以通过如下的命令来完成：
```
$ sudo apt install mysql-server
$ sudo apt-get install mysql-client
$ mysql_secure_installation
```

安装之后，通过如下命令可以查看 MySQL 服务器占用的端口：
```
$ sudo lsof -i -P
COMMAND     PID        USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
. . . . . .
mysqld    17860       mysql   20u  IPv4 136588      0t0  TCP localhost:3306 (LISTEN)
```
可以看到 MySQL 服务器在 TCP 的 ***3306*** 端口监听请求。

## 新MySQL建用户
安装好了 MySQL 之后，我们还需要创建用户，以便于后面在代码里使用。创建用户的方法如下：
```
## 登录MYSQL
$ mysql -u root -p
Enter password:
## 创建用户
mysql> CREATE USER 'hanpfei'@'localhost' IDENTIFIED BY 'hanpfei';
## 刷新系统权限表
mysql>flush privileges;
```
这样就创建了一个名为 `hanpfei`，密码为 `hanpfei` 的用户。

然后登录一下。
```
mysql> exit;
$ mysql -u hanpfei -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 5.7.17-0ubuntu0.16.04.1 (Ubuntu)
. . . . . .
mysql> 
```

## 创建数据库并为用户授权
登录MYSQL（有ROOT权限）。我里我以 ROOT 身份登录.
```
$ mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 12
Server version: 5.7.17-0ubuntu0.16.04.1 (Ubuntu)
```

为用户创建一个数据库 (`staff_info_db`)
```
mysql>create database staff_info_db;
```

授权 `hanpfei` 用户拥有 `staff_info_db` 数据库的所有权限。
```
mysql>grant all privileges on staff_info_db.* to hanpfei@localhost identified by 'hanpfei';
```
刷新系统权限表
```
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

## 创建表
有了数据库，接下来就是创建表了。建表语句如下：
```
CREATE TABLE `userinfo1` (
    `uid` INT(10) NOT NULL AUTO_INCREMENT,
    `username` VARCHAR(64) NULL DEFAULT NULL,
    `departname` VARCHAR(64) NULL DEFAULT NULL,
    `created` DATE NULL DEFAULT NULL,
    PRIMARY KEY (`uid`)
);

CREATE TABLE `userdetail` (
    `uid` INT(10) NOT NULL DEFAULT '0',
    `intro` TEXT NULL,
    `profile` TEXT NULL,
    PRIMARY KEY (`uid`)
);
```

具体的建表可以通过如下得命令来完成：
```
$ mysql -u hanpfei -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 18
Server version: 5.7.17-0ubuntu0.16.04.1 (Ubuntu)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use staff_info_db;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> CREATE TABLE `userinfo` (
    ->     `uid` INT(10) NOT NULL AUTO_INCREMENT,
    ->     `username` VARCHAR(64) NULL DEFAULT NULL,
    ->     `departname` VARCHAR(64) NULL DEFAULT NULL,
    ->     `created` DATE NULL DEFAULT NULL,
    ->     PRIMARY KEY (`uid`)
    -> );
Query OK, 0 rows affected (0.80 sec)

mysql> CREATE TABLE `userdetail` (
    ->     `uid` INT(10) NOT NULL DEFAULT '0',
    ->     `intro` TEXT NULL,
    ->     `profile` TEXT NULL,
    ->     PRIMARY KEY (`uid`)
    -> );
Query OK, 0 rows affected (0.80 sec)
mysql> exit
Bye
```
这样我们需要的表就建好了。

## 用 Go 语言访问 MySQL
安装好了数据库，并创建了适当的用户、数据库以及数据库表之后，我们就可以开始用 Go 语言访问 MySQL了。然而，在实际写代码之前，还要先下载对应的驱动。Go 语言中支持 MySQL 的驱动目前比较多，我们以支持 Go 语言标准 `database/sql` 接口的 `github.com/Go-SQL-Driver/MySQL` 为例来看。我们先要下载这个包：
```
$ go get github.com/Go-SQL-Driver/MySQL
```

然后来看如何使用 `database/sql` 接口对数据库表进行增删改查操作：
```
package main

import (
	_ "github.com/Go-SQL-Driver/MySQL"
	"fmt"
	"database/sql"
)

func checkErr(err error)  {
	if err != nil {
		panic(err)
	}
}

func mysqlTest()  {
	db, err := sql.Open("mysql", "hanpfei:hanpfei@/staff_info_db?charset=utf8")
	checkErr(err)
	defer db.Close()

	// Insert
	stmt, err := db.Prepare("INSERT userinfo SET username=?,departname=?,created=?")
	checkErr(err)

	res, err := stmt.Exec("hanpfei", "R&D Department", "2016-05-23")
	checkErr(err)

	id, err := res.LastInsertId()
	checkErr(err)

	fmt.Println(id)

	// Update
	stmt, err = db.Prepare("UPDATE userinfo SET username=? where uid=?")
	checkErr(err)

	res, err = stmt.Exec("hanpfeiupdate", id)
	checkErr(err)

	affect, err := res.RowsAffected()
	checkErr(err)

	fmt.Println(affect)

	// Query
	rows, err := db.Query("SELECT * FROM userinfo")
	checkErr(err)

	for rows.Next() {
		var uid int
		var username string
		var department string
		var created string
		err = rows.Scan(&uid, &username, &department, & created)
		checkErr(err)
		fmt.Println(uid)
		fmt.Println(username)
		fmt.Println(department)
		fmt.Println(created)
	}

	// Delete data
	stmt, err = db.Prepare("DELETE from userinfo where uid=?")
	checkErr(err)

	res, err = stmt.Exec(id)
	checkErr(err)

	affect, err = res.RowsAffected()
	checkErr(err)

	fmt.Println(affect)

	checkErr(err)
}

func main()  {
	mysqlTest()
}
```
通过以上代码，可以看出来 Go 语言操作 MySQL 数据库还是非常方便的。关于这个数据库驱动的用法更详细的信息，可以参考其 [官方文档](https://godoc.org/github.com/go-sql-driver/mysql)。

# SQLite3
SQLite 是一个开源的嵌入式系统的关系型数据库，它是实现自包容、零配置且支持事务的 SQL 数据库引擎。其特点是高度便携、结构紧凑、高效且可靠。与其它许多数据库相比，SQLite 的安装运行都非常简单。大多数情况下，只要确保 SQLite 的二进制文件存在，即可创建和访问数据库。

可以通过如下的命令来安装 SQLite：
```
$ sudo apt-get install libsqlite3-0 libsqlite3-dev
```
SQLite 的命令行工具可装可不装，不影响我们通过代码操作 SQLite 数据库。

同样我们需要选择一款 SQLite 驱动。Go 语言支持 SQLite 的驱动也比较多，但好多都不支持 `database/sql` 接口。这里我们选择支持 `database/sql` 接口的 `github.com/mattn/go-sqlite3`。

在开始写代码前，要先安装这个驱动：
```
$ go get github.com/mattn/go-sqlite3
```

建表的语句如下：
```
CREATE TABLE `userinfo` (
    `uid` INTEGER PRIMARY KEY AUTOINCREMENT,
    `username` VARCHAR(64) NULL,
    `departname` VARCHAR(64) NULL,
    `created` DATE NULL
);

CREATE TABLE `userdetail` (
    `uid` INT(10) NULL,
    `intro` TEXT NULL,
    `profile` TEXT NULL,
    PRIMARY KEY (`uid`)
);
```

不过我们同样通过代码来完成建表了。然后来看在 Go 语言中，具体如何操作 SQLite ：
```
package main

import (
	_ "github.com/mattn/go-sqlite3"
	"database/sql"
	"fmt"
)

func checkErr(err error)  {
	if err != nil {
		panic(err)
	}
}

func testOperationForSqlite3(db *sql.DB)  {
	// Insert
	stmt, err := db.Prepare("INSERT INTO userinfo(username, departname, created) values(?,?,?)")
	checkErr(err)

	res, err := stmt.Exec("hanpfei", "R&D Department", "2016-05-23")
	checkErr(err)

	id, err := res.LastInsertId()
	checkErr(err)

	fmt.Println(id)

	// Update
	stmt, err = db.Prepare("UPDATE userinfo SET username=? where uid=?")
	checkErr(err)

	res, err = stmt.Exec("hanpfeiupdate", id)
	checkErr(err)

	affect, err := res.RowsAffected()
	checkErr(err)

	fmt.Println(affect)

	// Query
	rows, err := db.Query("SELECT * FROM userinfo")
	checkErr(err)

	for rows.Next() {
		var uid int
		var username string
		var department string
		var created string
		err = rows.Scan(&uid, &username, &department, & created)
		checkErr(err)
		fmt.Println(uid)
		fmt.Println(username)
		fmt.Println(department)
		fmt.Println(created)
	}

	// Delete data
	stmt, err = db.Prepare("DELETE from userinfo where uid=?")
	checkErr(err)

	res, err = stmt.Exec(id)
	checkErr(err)

	affect, err = res.RowsAffected()
	checkErr(err)

	fmt.Println(affect)

	checkErr(err)
}

func createTableForSqlite3(db *sql.DB)  {
	createTableCommand := "CREATE TABLE userinfo (uid INTEGER PRIMARY KEY AUTOINCREMENT, username VARCHAR(64) NULL, " +
		"departname VARCHAR(64) NULL, created DATE NULL);"

	stmt, err := db.Prepare(createTableCommand)
	checkErr(err)

	_, err = stmt.Exec()
	checkErr(err)

	createTableCommand = "CREATE TABLE userdetail (uid INT(10) NULL, intro TEXT NULL, profile TEXT NULL, PRIMARY KEY (uid));"

	stmt, err = db.Prepare(createTableCommand)
	checkErr(err)

	_, err = stmt.Exec()
	checkErr(err)
}

func sqlite3Test()  {
	db, err := sql.Open("sqlite3", "./foo.db")
	checkErr(err)
	defer db.Close()

	createTableForSqlite3(db)
	testOperationForSqlite3(db)
}

func main()  {
	sqlite3Test()
}

```
执行上面的代码，将看到如下的输出：
```
/usr/lib/go/bin/go run /home/hanpfei0306/IdeaProjects/HelloGo/Sqlite3.go
1
1
1
hanpfeiupdate
R&D Department
2016-05-23T00:00:00Z
1

Process finished with exit code 0
```
我们可以看到，上面操作 SQLite 数据库的代码，和前面操作 MySQL 数据库的代码几乎一模一样。仅有的改变是导入的驱动变了，然后调用 `sql,Open()` 打开数据库驱动的方式不同。

# PostgreSQL
PostgreSQL 是一个开源的自由的对象-关系数据库服务器，它以灵活的 BSD-Style 许可发行。相对于其它许多的开源数据库系统（如 MySQL 和 Firebird）和商业数据库系统（如 Oracle 和 MS SQL Server），它为我们提供了另外的一种选择。

PostgreSQL 相对于 MySQL 而言更加庞大，因为它本是为替代 Oracle 而设计。

这里我们就来看一下如何用 Go 语言操作 PostgreSQL。

## 安装
我们首先要安装 PostgreSQL 服务器。使用如下命令，会自动安装最新版，这里为 `9.5`：
```
sudo apt-get install postgresql
```
安装完成后，默认会：
（1）创建名为 "postgres" 的Linux用户
（2）创建名为 "postgres"、不带密码的默认数据库账号作为数据库管理员
（3）创建名为 "postgres" 的表

安装完成后的一些默认信息如下：
```
config /etc/postgresql/9.5/main 
data /var/lib/postgresql/9.5/main 
locale en_US.UTF-8 
socket /var/run/postgresql 
port 5432
```

通过 `lsof -i` 命令我们可以查看 PostgreSQL 服务器占用的端口，并确认其在正常运行：
```
$ sudo lsof -i -P
. . . . . .
postgres  24581    postgres    6u  IPv6 284716      0t0  TCP localhost:5432 (LISTEN)
postgres  24581    postgres    7u  IPv4 284717      0t0  TCP localhost:5432 (LISTEN)
postgres  24581    postgres   11u  IPv6 279448      0t0  UDP localhost:42368->localhost:42368 
postgres  24583    postgres   11u  IPv6 279448      0t0  UDP localhost:42368->localhost:42368 
postgres  24584    postgres   11u  IPv6 279448      0t0  UDP localhost:42368->localhost:42368 
postgres  24585    postgres   11u  IPv6 279448      0t0  UDP localhost:42368->localhost:42368 
postgres  24586    postgres   11u  IPv6 279448      0t0  UDP localhost:42368->localhost:42368 
postgres  24587    postgres   11u  IPv6 279448      0t0  UDP localhost:42368->localhost:42368
```
可以看到 PostgreSQL 服务器在 TCP 的 ***5432*** 端口上监听请求，并开启了多个进程来监听。

## 配置
安装完后会有 PostgreSQL 的客户端 psql 可以用，通过 `sudo -u postgres psql` 进入，提示符变成 `postgres=# `：
```
$ sudo -u postgres psql
psql (9.5.6)
Type "help" for help.

postgres=# 
```

在这里可以执行 SQL 语句和 psql 的基本命令。

## 新建 PostgreSQL用户
我们可以通过如下的命令
```
postgres=# CREATE USER hanpfei PASSWORD 'hanpfei' CREATEDB;
CREATE ROLE
```
通过 `\du` 可以查看当前已经存在的用户列表：

```
postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 hanpfei   | Create DB                                                  | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

此时我们以新创建的用户登录PostgreSQL，会报出如下得error：
```
$ sudo -u postgres psql -U hanpfei
psql: FATAL:  Peer authentication failed for user "hanpfei"
```

我们可以通过修改 `pg_hba.conf` 文件来解决这个问题：
```
$ sudo gedit /etc/postgresql/9.5/main/pg_hba.conf
```
将其中 ***除 `postgres` 用户外*** 的所有用户的 `METHOD` 都从 `peer` 或 `ident` 改为 `md5` 或 `trust`。

修改完毕，保存退出。

然后执行如下命令重新加载配置：
```
$ /etc/init.d/postgresql reload
```

再次尝试用新创建的用户的登录 PostgreSQL 时报出了新的 error：
```
$ sudo -u postgres psql -U hanpfei
Password for user hanpfei: 
psql: FATAL:  database "hanpfei" does not exist
```
提示用户的数据库不存在。我们还需要再次登录 `postgres` 用户，为我们的新用户创建数据库：
```
$ sudo -u postgres psql
psql (9.5.6)
Type "help" for help.

postgres=# create database hanpfei;
CREATE DATABASE
postgres=# \q
```

然后就可以以新用户登录了：
```
$ sudo -u postgres psql -U hanpfei
[sudo] hanpfei0306 的密码： 
Password for user hanpfei: 
psql (9.5.6)
Type "help" for help.

hanpfei=> 
```
这里我们需要输入两次密码，一次是当前 Linux 用户的用户密码，用来执行 sudo，另一次是数据库的用户的密码，用来登录数据库。

接着我们创建一个新的应用数据库，并选择它作为我们当前使用的数据库，这可以通过 `\c `命令来实现：
```
hanpfei=> \l
                                    List of databases
     Name      |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
---------------+----------+----------+-------------+-------------+-----------------------
 hanpfei       | postgres | UTF8     | zh_CN.UTF-8 | zh_CN.UTF-8 | 
 postgres      | postgres | UTF8     | zh_CN.UTF-8 | zh_CN.UTF-8 | 
 staff_info_db | hanpfei  | UTF8     | zh_CN.UTF-8 | zh_CN.UTF-8 | 
 template0     | postgres | UTF8     | zh_CN.UTF-8 | zh_CN.UTF-8 | =c/postgres          +
               |          |          |             |             | postgres=CTc/postgres
 template1     | postgres | UTF8     | zh_CN.UTF-8 | zh_CN.UTF-8 | =c/postgres          +
               |          |          |             |             | postgres=CTc/postgres
(5 rows)

hanpfei=> create database staff_info_db;

hanpfei=> \c staff_info_db;
You are now connected to database "staff_info_db" as user "hanpfei".

staff_info_db=> 
```
上面的 `\l` 是用来列出当前已创建的所有数据库的。注意我们切换了数据库之后，命令输入提示字符串也变为了新数据库的名字。

然后创建表。建表语句如下：

```
CREATE TABLE userinfo (
    uid serial NOT NULL,
    username character varying(100) NOT NULL,
    departname character varying(100) NOT NULL,
    created date,
    CONSTRAINT userinfo_pkey PRIMARY KEY (uid)
)
WITH (OIDS=FALSE);

CREATE TABLE userdetail (
    uid integer,
    intro character varying(100),
    profile character varying(100)
)
WITH (OIDS=FALSE);
```
具体命令则是：
```
staff_info_db=> CREATE TABLE userinfo (
staff_info_db(>     uid serial NOT NULL,
staff_info_db(>     username character varying(100) NOT NULL,
staff_info_db(>     departname character varying(100) NOT NULL,
staff_info_db(>     created date,
staff_info_db(>     CONSTRAINT userinfo_pkey PRIMARY KEY (uid)
staff_info_db(> )
staff_info_db-> WITH (OIDS=FALSE);
CREATE TABLE
staff_info_db=> CREATE TABLE userdetail (
staff_info_db(>     uid integer,
staff_info_db(>     intro character varying(100),
staff_info_db(>     profile character varying(100)
staff_info_db(> )
staff_info_db-> WITH (OIDS=FALSE);
CREATE TABLE
staff_info_db=>
```

## 权限
创建的表还可以赋予其它用户完全的权限，比如，以 `postgres` 用户创建的表的完全的权限赋予新用户：
```
postgres=# GRANT ALL ON userinfo TO hanpfei;
postgres=# GRANT ALL ON userinfo TO userdetail;
```

## 使用 Go 语言操作 PostgreSQL
同样我们需要先找个驱动。Go 语言的 PostgreSQL 驱动也很多。这里我们选用支持 `database/sql` 接口的驱动 `github.com/lib/pq`，这个项目由 `github.com/bmizerany/pq` 迁移而来，而后者目前已经废弃。我们还是要先安装：
```
$ go get github.com/lib/pq
```

然后用 Go 语言访问 PostgreSQL：
```
package main

import (
	_ "github.com/lib/pq"
	"fmt"
	"database/sql"
)

func checkErr(err error)  {
	if err != nil {
		panic(err)
	}
}

func testOperationForPostgreSQL(db *sql.DB)  {
	// Insert
	stmt, err := db.Prepare("INSERT INTO userinfo(username,departname,created) VALUES($1,$2,$3) RETURNING uid")
	checkErr(err)

	res, err := stmt.Exec("hanpfei", "R_D_Department", "2016-05-23")
	checkErr(err)

	// id, err := res.LastInsertId()
	// checkErr(err)

	fmt.Println(res)

	// Update
	stmt, err = db.Prepare("UPDATE userinfo SET username=$1 where uid=$2")
	checkErr(err)

	res, err = stmt.Exec("hanpfeiupdate", 1)
	checkErr(err)

	affect, err := res.RowsAffected()
	checkErr(err)

	fmt.Println(affect)

	// Query
	rows, err := db.Query("SELECT * FROM userinfo")
	checkErr(err)

	for rows.Next() {
		var uid int
		var username string
		var department string
		var created string
		err = rows.Scan(&uid, &username, &department, & created)
		checkErr(err)
		fmt.Println(uid)
		fmt.Println(username)
		fmt.Println(department)
		fmt.Println(created)
	}

	// Delete data
	stmt, err = db.Prepare("DELETE from userinfo where uid=$1")
	checkErr(err)

	res, err = stmt.Exec(1)
	checkErr(err)

	affect, err = res.RowsAffected()
	checkErr(err)

	fmt.Println(affect)

	checkErr(err)
}

func postgreSqlTest()  {
	db, err := sql.Open("postgres", "user=hanpfei password=hanpfei dbname=staff_info_db sslmode=disable")
	checkErr(err)
	defer db.Close()

	testOperationForPostgreSQL(db)
}

func main()  {
	postgreSqlTest()
}

```
从上面的代码中可以看到，PostgreSQL 是通过 “$1，$2”这种方式来占位要传入的参数的，而不是 MySQL 中的 “?” 另外在 `sql.Open()`中的 dsn 信息的格式也与 MySQL 不同，因而在使用时需要注意。
此外， PostgreSQL 不支持 `LastInsertId()` 函数，因为它内部没有实现类似 MySQL 的自增 ID 返回。其它的代码则几乎一模一样。

# NoSQL 数据库

NoSQL （Not only SQL），指的是非关系型数据库。随着 Web 2.0 的兴起，传统的关系型数据库在应付新应用，特别是超大规模和高并发得 SNS 型 Web 2.0 纯动态网站时已经显得力不从心，暴露了很多难以客服得问题，而非关系型数据库则由于其自身的有点，而得到迅速得发展。

在看了上面的几种 SQL 数据库之后，接下来我们将学习用 Go 语言操作几种 NoSQL 数据库，主要是 Redis 和 MongoDB。

# Redis
Redis 是一个开源的（BSD 许可），基于内存的数据结构存储产品，它可以被用作数据库，缓存和消息代理。它支持的数据结构非常多，如 string（字符串），hash（哈希），list（列表），set（集合），带有范围查询的sorted set（有序集合），bitmap（位图），超文本和具有半径查询的地理空间索引。Redis具有内置复制，Lua脚本，LRU驱逐，事务和不同级别的磁盘持久性，并通过Redis Sentinel提供高可用性，并通过Redis Cluster进行自动分区。

Redis 是一个超高性能的 key-value 数据库。Redis 的出现，很大程度补偿了memcached 这类 key-value 存储的不足，在部 分场合可以对关系数据库起到很好的补充作用。它提供了Python，Ruby，Erlang，PHP客户端，使用很方便。

[Redis 主页](https://redis.io)。

[Redis GitHub主页](https://github.com/antirez/redis)。

这里我们就来看一下如何用 Go 语言操作Redis。

## Redis 安装
使用如下命令，会自动安装最新版
```
$ sudo apt-get install redis-server redis-tools
```
通过 `lsof -i` 命令我们可以查看 Redis 服务器占用的端口。
```
$ sudo lsof -i -P
COMMAND     PID        USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
. . . . . .
redis-ser  2977       redis    4u  IPv4 507101      0t0  TCP localhost:6379 (LISTEN)
```
可以看到 Redis 服务器在 TCP 的 ***6379*** 端口监听请求。

Redis 还提供了功能强大的命令行工具 
```
$ redis-cli
127.0.0.1:6379> help
redis-cli 3.0.6
Type: "help @<group>" to get a list of commands in <group>
      "help <command>" for help on <command>
      "help <tab>" to get a list of possible help topics
      "quit" to exit
```

## Go 程序访问 Redis
Go 语言的 Redis 客户端驱动还是很多的，具体可以通过 [Redis 的 Client 页](https://redis.io/clients#go) 找到它们。当前还处于比较活跃的状态，也就是近 6 个月官方 repo 有过更新的驱动如下：
```
https://github.com/go-redis/redis
https://github.com/keimoon/gore
https://github.com/gosexy/redis
https://github.com/tideland/golib
https://github.com/garyburd/redigo
https://github.com/mediocregopher/radix.v2
```
其中最后两个，是目前 Redis 官方推荐使用的 Go 驱动。这里我们以最后一个为例，来看要如何在 Go 代码里操作 Redis。然而，在开始之前，我们还是要先安装相应的包：
```
$ go get github.com/mediocregopher/radix.v2/redis
```

接着来看具体如何通过 Go 语言访问 Redis：
```
package main

import (
	"fmt"
	"github.com/mediocregopher/radix.v2/redis"
)

func checkErr(err error)  {
	if err != nil {
		panic(err)
	}
}

func testRedisWithRadix()  {
	client, err := redis.Dial("tcp", "localhost:6379")
	checkErr(err)

	err = client.Cmd("SET", "a", "hello").Err
	checkErr(err)

	val, err := client.Cmd("GET", "a").Str()
	checkErr(err)

	fmt.Println("Redis val:")
	fmt.Println(string(val))

	err = client.Cmd("DEL", "a").Err
	checkErr(err)

	// list operation
	vals := []string{"a", "b", "c", "d", "e"}
	for _, v := range vals {
		err = client.Cmd("RPUSH", "l", v).Err
		checkErr(err)
	}
	dbvals, err := client.Cmd("LRANGE", "l", 0, 4).List()
	checkErr(err)
	for i, v := range dbvals  {
		fmt.Println(i, ":", string(v))
	}
	err = client.Cmd("DEL", "l").Err
	checkErr(err)
}

func main()  {
	testRedisWithRadix()
}
```

运行上面代码，可以看到如下的输出：
```
$ /usr/lib/go/bin/go run /home/hanpfei0306/IdeaProjects/HelloGo/Redis.go
Redis val:
hello
0 : a
1 : b
2 : c
3 : d
4 : e

Process finished with exit code 0
```
我们可以看到，操作 Redis 非常方便，大多只要用 `client` 命令即可。关于 `Radix` 用法的更多详细内容，可以参考其 [官方文档](https://godoc.org/github.com/mediocregopher/radix.v2/redis)。

# MongoDB
MongoDB 是一个高性能功能强大而流行的分布式文档存储 NoSQL (Not Only SQL) 数据库产品。它是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。他支持的数据结构非常松散，是类似json的bjson格式，因此可以存储比较复杂的数据类型。Mongo最大的特点是他支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能，而且还支持对数据建立索引。

下面我们来看一下如何通过 Go 语言操作 MongoDB。

## MongoDB 安装
首先需要安装 MongoDB。使用如下命令，会自动安装最新版 MongoDB，这里是 `2.6.10` 版。
```
$ sudo apt-get install mongodb mongodb-clients mongodb-server
```

通过 `lsof -i` 命令我们可以查看 MongoDB 服务器占用的端口，并确认 MongoDB 服务器的正常运行。
```
 sudo lsof -i -P
COMMAND     PID        USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
. . . . . .
mongod     7838     mongodb    8u  IPv4 536811      0t0  TCP localhost:27017 (LISTEN)
```
可以看到 MongoDB 服务器在 TCP 的 ***27017*** 端口监听请求。

## 创建数据库
安装了 MongoDB 服务器之后，我们还要创建数据库，以备后面在代码里面用。我们可以通过 MongoDB 的 shell 版本，使用 `use`命令创建数据库：
```
$ mongo
MongoDB shell version: 2.6.10
connecting to: test
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	http://docs.mongodb.org/
Questions? Try the support group
	http://groups.google.com/group/mongodb-user
> use people_info
switched to db people_info
```
`use` 命令在 MySQL 中用于选择已经创建好的数据库，而在 MongoDB 中则会自动创建当前还不存在的数据库。

## Go 程序访问 MongoDB
在开始编写 Go 代码访问 MongoDB 数据库之前，我们还要先下载 MongoDB 的Go语言驱动。在 MongoDB 的 [官方 Drivers主页](https://docs.mongodb.com/ecosystem/drivers/go/) 可以找到可用的 Go 语言驱动。当前官方推荐的只有开源社区支持的 `mgo` 驱动。[mgo 主页](http://labix.org/mgo)。mgo 的 [GitHub 主页](https://github.com/go-mgo/mgo)。通过 `go get` 命令安装我们需要的两个 mgo Go 包：
```
$ go get gopkg.in/mgo.v2
$ go get gopkg.in/mgo.v2/bson
```

接着来看具体如何通过 Go 语言操作 MongoDB。
```
package main

import (
	"gopkg.in/mgo.v2"
	"gopkg.in/mgo.v2/bson"
	"fmt"
)

type Person struct {
	Name string
	Phone string
}

func checkErr(err error)  {
	if err != nil {
		panic(err)
	}
}

func testMongoDB()  {
	session, err := mgo.Dial("localhost")
	checkErr(err)
	defer session.Close()

	session.SetMode(mgo.Monotonic, true)
	c := session.DB("people_info").C("people")

	err = c.Insert(&Person{"Ale", "+55 53 8116 9639"},
		&Person{"Cla", "+55 53 8402 8510"})
	checkErr(err)

	result := Person{}
	err = c.Find(bson.M{"name": "Ale"}).One(&result)
	checkErr(err)

	fmt.Println("Phone:", result.Phone)
}

func main()  {
	testMongoDB()
}
```
操作与许多 SQL数据库的 ORM 库提供的很相似，可以直接操作对象。

# 参考文档：
[Ubuntu 16.04 安装 Apache, MySQL, PHP7](http://www.cnblogs.com/2016xt/p/5517049.html)

[Ubuntu PostgreSQL安装和配置](http://www.cnblogs.com/z-sm/p/5644165.html#autoid-2-1-0)

[postgresql 切换数据库命令](http://blog.csdn.net/nameoccupied/article/details/9145345)

[PostgreSQL选择数据库](http://outofmemory.cn/PostgreSQL/tutorial/PostgreSQL-select-data-library)

[PostgreSQL删除表](http://www.yiibai.com/html/postgresql/2013/080440.html)

[PostgreSQL学习手册(角色和权限)](http://www.cnblogs.com/stephen-liu74/archive/2012/05/18/2302639.html)

[PostgreSQL 数据库安装与配置](https://doc.odoo.com/6.1/zh_CN/install/linux/postgres/)

[reids客户端 redis-cli用法](http://www.ttlsa.com/redis/the-reids-client-redis-cli-using/)
