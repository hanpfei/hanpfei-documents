---
title: 为curl命令启用HTTP2支持
date: 2016-11-14 16:05:49
categories: HTTP2
---

## 检查curl版本

可以通过如下命令检查当前安装的curl支持的协议及特性：
```
$ curl --version
curl 7.47.0 (x86_64-pc-linux-gnu) libcurl/7.47.0 GnuTLS/3.4.10 zlib/1.2.8 libidn/1.32 librtmp/2.3
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtmp rtsp smb smbs smtp smtps telnet tftp 
Features: AsynchDNS IDN IPv6 Largefile GSS-API Kerberos SPNEGO NTLM NTLM_WB SSL libz TLS-SRP UnixSockets
```
可以看到当前安装的curl支持的http、https等协议，及其它功能，但其中并没有包含HTTP2。

<!--more-->

此时我们强制curl以HTTP2向请求服务器请求服务，会看到如下的报错信息：
```
$ curl --http2 -v https://www.wolfcstech.com/2016/11/14/nginx_uWSGI_deply_django_on_ubuntu/
curl: (1) Unsupported protocol
```

## 编译安装nghttp2

curl依赖nghttp2提供对HTTP2的支持，因而首先需要安装nghttp2。

下载并安装nghttp2：
```
$ git clone git@github.com:nghttp2/nghttp2.git
$ cd nghttp2
$ ./configure
$ sudo make & make install
```

## 编译安装curl
下载、配置并安装curl
```
$ git clone git@github.com:curl/curl.git
$ cd curl
$ ./configure --with-nghttp2=/usr/local --with-ssl
$ make
$ sudo make install
```
还可以通过`./configure --help`查看可以为curl做的其它配置：
```
hanpfei0306@ThundeRobot:~/data/osprojects/curl$ ./configure --help
`configure' configures curl - to adapt to many kinds of systems.

Usage: ./configure [OPTION]... [VAR=VALUE]...

To assign environment variables (e.g., CC, CFLAGS...), specify them as
VAR=VALUE.  See below for descriptions of some of the useful variables.

Defaults for the options are specified in brackets.

Configuration:
  -h, --help              display this help and exit
      --help=short        display options specific to this package
      --help=recursive    display the short help of all the included packages
  -V, --version           display version information and exit
  -q, --quiet, --silent   do not print `checking ...' messages
      --cache-file=FILE   cache test results in FILE [disabled]
  -C, --config-cache      alias for `--cache-file=config.cache'
  -n, --no-create         do not create output files
      --srcdir=DIR        find the sources in DIR [configure dir or `..']

Installation directories:
  --prefix=PREFIX         install architecture-independent files in PREFIX
                          [/usr/local]
  --exec-prefix=EPREFIX   install architecture-dependent files in EPREFIX
                          [PREFIX]

By default, `make install' will install all the files in
`/usr/local/bin', `/usr/local/lib' etc.  You can specify
an installation prefix other than `/usr/local' using `--prefix',
for instance `--prefix=$HOME'.

For better control, use the options below.

Fine tuning of the installation directories:
  --bindir=DIR            user executables [EPREFIX/bin]
  --sbindir=DIR           system admin executables [EPREFIX/sbin]
  --libexecdir=DIR        program executables [EPREFIX/libexec]
  --sysconfdir=DIR        read-only single-machine data [PREFIX/etc]
  --sharedstatedir=DIR    modifiable architecture-independent data [PREFIX/com]
  --localstatedir=DIR     modifiable single-machine data [PREFIX/var]
  --runstatedir=DIR       modifiable per-process data [LOCALSTATEDIR/run]
  --libdir=DIR            object code libraries [EPREFIX/lib]
  --includedir=DIR        C header files [PREFIX/include]
  --oldincludedir=DIR     C header files for non-gcc [/usr/include]
  --datarootdir=DIR       read-only arch.-independent data root [PREFIX/share]
  --datadir=DIR           read-only architecture-independent data [DATAROOTDIR]
  --infodir=DIR           info documentation [DATAROOTDIR/info]
  --localedir=DIR         locale-dependent data [DATAROOTDIR/locale]
  --mandir=DIR            man documentation [DATAROOTDIR/man]
  --docdir=DIR            documentation root [DATAROOTDIR/doc/curl]
  --htmldir=DIR           html documentation [DOCDIR]
  --dvidir=DIR            dvi documentation [DOCDIR]
  --pdfdir=DIR            pdf documentation [DOCDIR]
  --psdir=DIR             ps documentation [DOCDIR]

Program names:
  --program-prefix=PREFIX            prepend PREFIX to installed program names
  --program-suffix=SUFFIX            append SUFFIX to installed program names
  --program-transform-name=PROGRAM   run sed PROGRAM on installed program names

System types:
  --build=BUILD     configure for building on BUILD [guessed]
  --host=HOST       cross-compile to build programs to run on HOST [BUILD]

Optional Features:
  --disable-option-checking  ignore unrecognized --enable/--with options
  --disable-FEATURE       do not include FEATURE (same as --enable-FEATURE=no)
  --enable-FEATURE[=ARG]  include FEATURE [ARG=yes]
  --enable-maintainer-mode
                          enable make rules and dependencies not useful (and
                          sometimes confusing) to the casual installer
  --enable-silent-rules   less verbose build output (undo: "make V=1")
  --disable-silent-rules  verbose build output (undo: "make V=0")
  --enable-debug          Enable debug build options
  --disable-debug         Disable debug build options
  --enable-optimize       Enable compiler optimizations
  --disable-optimize      Disable compiler optimizations
  --enable-warnings       Enable strict compiler warnings
  --disable-warnings      Disable strict compiler warnings
  --enable-werror         Enable compiler warnings as errors
  --disable-werror        Disable compiler warnings as errors
  --enable-curldebug      Enable curl debug memory tracking
  --disable-curldebug     Disable curl debug memory tracking
  --enable-symbol-hiding  Enable hiding of library internal symbols
  --disable-symbol-hiding Disable hiding of library internal symbols
  --enable-hidden-symbols To be deprecated, use --enable-symbol-hiding
  --disable-hidden-symbols
                          To be deprecated, use --disable-symbol-hiding
  --enable-ares[=PATH]    Enable c-ares for DNS lookups
  --disable-ares          Disable c-ares for DNS lookups
  --disable-rt            disable dependency on -lrt
  --enable-dependency-tracking
                          do not reject slow dependency extractors
  --disable-dependency-tracking
                          speeds up one-time build
  --disable-largefile     omit support for large files
  --enable-shared[=PKGS]  build shared libraries [default=yes]
  --enable-static[=PKGS]  build static libraries [default=yes]
  --enable-fast-install[=PKGS]
                          optimize for fast installation [default=yes]
  --disable-libtool-lock  avoid locking (might break parallel builds)
  --enable-http           Enable HTTP support
  --disable-http          Disable HTTP support
  --enable-ftp            Enable FTP support
  --disable-ftp           Disable FTP support
  --enable-file           Enable FILE support
  --disable-file          Disable FILE support
  --enable-ldap           Enable LDAP support
```
再次检查curl的版本及其支持的功能列表：
```
$ curl --version
curl 7.50.2-DEV (x86_64-pc-linux-gnu) libcurl/7.50.2-DEV OpenSSL/1.0.2g zlib/1.2.8 nghttp2/1.14.0-DEV
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtsp smb smbs smtp smtps telnet tftp 
Features: IPv6 Largefile NTLM NTLM_WB SSL libz TLS-SRP HTTP2 UnixSockets
```
已经可以看到HTTP2。

## 强制curl以HTTP2请求资源

可以以如下这样的命令强制curl以HTTP2协议向服务器请求服务：
```
$ curl --http2 -v https://www.wolfcstech.com/2016/11/14/nginx_uWSGI_deply_django_on_ubuntu/
*   Trying 139.196.224.72...
* TCP_NODELAY set
* Connected to www.wolfcstech.com (139.196.224.72) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: none
* TLSv1.2 (OUT), TLS header, Certificate Status (22):
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=www.wolfcstech.com
*  start date: Oct 25 01:12:00 2016 GMT
*  expire date: Jan 23 01:12:00 2017 GMT
*  subjectAltName: host "www.wolfcstech.com" matched cert's "www.wolfcstech.com"
*  issuer: C=US; O=Let's Encrypt; CN=Let's Encrypt Authority X3
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7a86b0)
> GET /2016/11/14/nginx_uWSGI_deply_django_on_ubuntu/ HTTP/1.1
> Host: www.wolfcstech.com
> User-Agent: curl/7.50.2-DEV
> Accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 200 
< server: nginx
< date: Mon, 14 Nov 2016 09:15:02 GMT
< content-type: text/html
< content-length: 68614
< last-modified: Mon, 14 Nov 2016 03:55:02 GMT
< etag: "58293596-10c06"
< accept-ranges: bytes
< 
<!doctype html>



  


<html class="theme-next mist use-motion">
<head>
  <meta charset="UTF-8"/>
<meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1"/>
```

## 参考文档
[如何启用curl命令HTTP2支持](https://www.sysgeek.cn/curl-with-http2-support/)
Done。
