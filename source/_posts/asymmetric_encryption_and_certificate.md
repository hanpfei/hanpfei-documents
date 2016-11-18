---
title: 非对称加密与证书
date: 2016-11-18 13:17:49
tags: 安全 HTTPS
---

﻿SSL/TLS是一个密码学协议，它的目标并不仅仅是网页内容的加密传输。SSL/TLS的主要目标有四个：加密安全、互操作性、可扩展性和效率。对于安全性的保障，它还会从多个方面进行，包括机密性，真实性以及完整性。机密性是指，传输的内容不被除通信的双方外的第三方获取；真实性是指，通信的对端正是期待的对端，而不是其它第三方冒充的；完整性则是指，传输的数据是完整的，数据没有被篡改或丢失。为了平衡多种需求，SSL/TLS被设计为一个安全框架，其中可以应用多种不同的安全方案，每种方案都由多个复杂的密码学过程组成。不同的安全方案，在安全性和效率之间有着不同的取舍，并由不同的密码学过程组成。

<!--more-->

在密码学上，非对称加密具有更高的安全性，同时计算复杂度更高，性能更差；而对称加密，则效率比较高，计算复杂度较低，但如果在通信过程中明文传输密钥，或将密钥以hard code的形式写在代码里，则安全隐患比较大。

从密码学过程的特性出发，整体来看，SSL/TLS连接是在会话协商阶段，通过 **非对称加密** 算法，如RSA、ECDHE等，完成身份验证，及后续用到的对称加密密钥的交换；在整个数据传输阶段，通过对称加密算法，如AES、3DES等，对传输的数据进行加密；通过数据散列算法，如SHA1、SHA256等，计算数据的散列值并随应用数据一起发送，以保证数据的完整性。

本文主要描述非对称加密的基本思想，及TLS的证书身份认证。

# 非对称加密
非对称加密 (asymmetric encryption) 又称为公钥加密 (public key cryptography)，是使用两个密钥，而不像对称加密那样使用一个密钥的加密方法；其中一个密钥是私密的，另一个是公开的。顾名思义，一个密钥用于非公开的私人的，另一个密钥将会被所有人共享。

**非对称加密的两个密钥之间存在一些特殊的数学关系，使得密钥具备一些有用的特性。如果利用某人的公钥加密数据，那么只有他们对应的私钥能够解密。从另一方面讲，如果某人用私钥加密数据，任何人都可以利用对应的公钥解开消息。** 后面这种操作不提供机密性，但可以用作数字签名。

## 非对称加密保护数据安全

盗用阮一峰老师的几幅图来说明，通过非对称加密保护数据安全的过程。

 - 鲍勃有两把钥匙，一把是公钥，另一把是私钥。

![](http://upload-images.jianshu.io/upload_images/1315506-4bdc062fc2e04eb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 鲍勃把公钥送给他的朋友们----帕蒂、道格、苏珊----每人一把。

![](http://upload-images.jianshu.io/upload_images/1315506-846117a1cc6173a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 苏珊要给鲍勃写一封不希望别人看到的保密的信。她写完后用鲍勃的公钥加密，就可以达到保密的效果。

![](http://upload-images.jianshu.io/upload_images/1315506-f7da60bdd7ba832f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 鲍勃收到信后，用私钥解密，就可以看到信件内容。

![](http://upload-images.jianshu.io/upload_images/1315506-97125d541d70613a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于非对称加密，**只要私钥不泄露**，传输的数据就是安全的。即使数据被别人获取，也无法解密。

非对称加密的这些特性直击对称加密中只用一个密钥，而该密钥不方便传输、保存的痛点。它大大方便了大规模团体的安全通信，方便了安全通信在互联网中的应用。

## 非对称加密生成数字签名

数据的加密安全只是数据安全的一个方面，数据的真实性同样非常重要。经常可以看到这样的案例，骗子在同学参加四、六级考试的时候，给同学的家长打电话或发短信，声称自己是学校的辅导员，并表示同学病重急需用钱，要求家长汇钱，同学家长汇钱给骗子而遭受巨大损失的情况。这就是数据/信息真实性没有得到足够验证而产生的问题。

再比如，一个仿冒的taobao网站，域名与真实的网站非常相似。我们一不小心输错了域名，或域名被劫持而访问了这个仿冒的网站，然后像平常在taobao购物一样，选择宝贝，并付款，但最后却怎么也收不到货物。

现实世界中，常常会请消息的发送者在消息后面签上自己的名字，或者印章来证明消息的真实可靠性，如信件中的签名，合同中的印章等等。类似地，在虚拟的网络世界中，也会通过数字签名来确认数据的真实可靠性。数字签名依赖的主要算法也是非对称加密，生成数字签名主要是使用私钥加密 **数据的散列摘要** 来签名。

通过几幅图来说明这个过程。
 - 鲍勃给苏珊回信，决定采用"数字签名"来证明自己的身份，表示自己对信件的内容负责。他写完后先用散列函数，生成信件的摘要（digest）。

![](http://upload-images.jianshu.io/upload_images/1315506-4162736fa50ddf8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 然后，鲍勃使用自己的私钥，对这个摘要加密，生成"数字签名"（signature）。

![](http://upload-images.jianshu.io/upload_images/1315506-8bcc4f094b3c6255.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 鲍勃将这个签名，附在信件下面，一起发给苏珊。

![](http://upload-images.jianshu.io/upload_images/1315506-4af9ba2ccc7dc241.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 苏珊收到信后，取下数字签名，用鲍勃的公钥解密，得到信件的摘要。

![](http://upload-images.jianshu.io/upload_images/1315506-9fe9f0a77032433f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 苏珊再对信件本身应用散列函数，将得到的结果，与上一步得到的摘要进行对比。如果两者一致，则证明这封信确实是鲍勃发出的，信件完整且未被篡改过。

![](http://upload-images.jianshu.io/upload_images/1315506-c7e39a037d89f203.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果鲍勃向苏珊借了钱，并用上面这样的过程写信给苏珊确认自己收到了钱，那么鲍勃就再也不能抵赖了——信件的末尾可是清清楚楚地签着鲍勃的大名呢。

如果我们的网络世界能像上图的过程那样，每个人都可以方便地获得可靠的公钥，那就太美好了。互联网上的网站成千上万，每个人都走到自己要访问的网站站长的办公室把网站的公钥拷走，或者网站站长挨个走到自己的目标用户家门口，把自己网站的公钥交给用户，那可就太麻烦，代价太大了。

公钥通常都是通过网络传输的。不怀好意的人，会试图干扰这个传输过程，将自己伪造的公钥发送给用户，进而破坏后续整个数据传输的安全性。如果用户拿到的是伪造的公钥，那签名也就形同虚设。

如道格想欺骗苏珊，他在鲍勃将公钥交给苏珊时拦住鲍勃，表示要替鲍勃转交。正好鲍勃有老板交待的其它重要事情要完成，于是就把自己的公钥交给道格请他帮忙转交。但道格把鲍勃的公钥丢进垃圾桶，而把自己的公钥交给了苏珊。此时，苏珊实际拥有的是道格的公钥，但还以为这是鲍勃的。因此，道格就可以冒充鲍勃，用自己的私钥做成"数字签名"，写信给苏珊，让苏珊用假的鲍勃公钥进行解密。

![](http://upload-images.jianshu.io/upload_images/1315506-ffdb2b2f36579bb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 证书

证书正是为了解决公钥的信任问题而引入。证书体系通过引入可信的第三机构，称为 **证书签发机构（certificate authority，简称CA）**，为公钥做认证，来确保公钥的真实可靠。证书是经过了 **CA** 私钥签名的 **证书持有者的公钥、身份信息及其它相关信息** 的文件，用户通过 **CA** 的公钥解密获取证书中包含的 **证书持有者** 的公钥。只要 **CA** 的私钥保持机密，通过证书验证 **证书持有者** 的身份，及获取公钥的过程就可靠。

## 证书的工作过程

互联网PKI证书体系的结构如下图：

![](http://upload-images.jianshu.io/upload_images/1315506-0573ac1556ca9a62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 **证书订阅人** ，也就是需要提供安全服务的人，向 **证书中心 (CA)** 的代理—— **登记机构** 提交自己的公钥来申请证书。 **登记机构** 对 **证书订阅人** 的身份进行核实，然后向 **证书中心 (CA)** 提交 **证书订阅人** 的公钥及身份信息。 **证书中心 (CA)** 用自己的私钥，对 **证书订阅人** 的公钥、身份信息和其它一些相关信息进行加密，生成 **"数字证书"（Digital Certificate）** ，并发送给 **登记机构**。 **登记机构** 将证书发送给 **证书订阅人** 。 **证书订阅人** 将证书部署在Web服务器上。 **信赖方**，即安全服务的用户，维护 **CA** 根证书库，并在与Web服务器通信时，从服务器获得证书。 **信赖方** 用CA根证书验证接收到的证书的有效性，进而验证服务器的真实性。

同样通过几幅图来说明这个过程。

 - 鲍勃去找证书签发机构，为公钥做认证。证书中心用自己的私钥，对鲍勃的公钥、身份信息和一些其它相关信息一起加密，生成"数字证书"（Digital Certificate）。

![](http://upload-images.jianshu.io/upload_images/1315506-043f723024c31644.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 鲍勃拿到数字证书以后，就可以放心，以后再也没人能冒充自己了。再需要发送自己的公钥给朋友们时，只要把自己事先拿到的 **数字证书** 发送给朋友就可以了。需要写信给苏珊时，照常附上自己的数字签名即可。

![](http://upload-images.jianshu.io/upload_images/1315506-66f43b8757962180.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 苏珊收到信后，用CA的公钥解开数字证书，就可以拿到鲍勃真实的公钥，然后就能证明"数字签名"是否真的是鲍勃签的。

![](http://upload-images.jianshu.io/upload_images/1315506-f51a67cbaaaea6fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 证书里有什么

PKI证书是 **抽象语法表示一 (Abstract syntax notation one, ASN.1)** 表示， **基本编码规则 (base encoding rules, BER)** 的一个子集 **唯一编码规则 (distinguished encoding rules, DER)** 编码的二进制文件。我们通常看到的证书则是DER使用Base64编码后的ASCII编码格式，即 **PEM (Privacy-enhanced mail)** 这种更容易发送、复制和粘贴的编码格式的纯文本文件。

证书里到底都有些什么呢？这里我们通过一个实际的证书来看一下。我们可以通过`openssl`解码数字证书，获得证书的可读形式：
```
$ openssl x509 -in chained.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            03:5c:25:82:1d:c2:b2:2f:6f:73:39:48:9c:68:07:1b:48:2d
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=Let's Encrypt, CN=Let's Encrypt Authority X3
        Validity
            Not Before: Oct 25 01:12:00 2016 GMT
            Not After : Jan 23 01:12:00 2017 GMT
        Subject: CN=www.wolfcstech.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:c3:92:70:78:ff:00:0a:22:c7:14:0b:3d:b3:26:
                    34:cb:37:63:26:1d:d6:42:7b:5c:ab:51:cc:f7:12:
                    57:26:b1:d1:4f:5f:a7:02:5b:3c:f3:e6:e1:ec:7c:
                    66:61:ba:d8:5e:d6:61:60:48:d6:d3:4c:23:9a:50:
                    75:4b:2d:1b:89:7d:7b:55:2f:12:63:b4:ac:c7:b5:
                    d1:44:95:ed:a2:f4:9d:ee:77:3c:2b:06:48:d9:18:
                    21:d1:ee:cf:5c:26:ad:c2:11:28:9c:27:65:11:94:
                    c4:1d:0f:5e:4c:4f:00:71:cf:5d:1f:40:4b:9a:5e:
                    3b:b0:42:45:c5:68:01:62:29:c2:92:b5:ad:8d:13:
                    11:db:7e:02:65:14:6a:5d:4b:66:16:08:d4:ab:90:
                    dc:06:28:27:cd:84:c0:b7:30:22:ff:54:71:c2:3b:
                    8d:7d:8b:52:c3:a8:f1:ee:63:42:2a:dd:4d:a7:70:
                    66:c5:c3:54:d5:8e:a1:e2:02:d0:8b:2f:f6:44:1d:
                    f5:f2:85:fd:49:c7:e0:d7:d0:ae:21:b7:25:ae:7c:
                    15:dc:56:51:45:f1:e7:19:d6:1c:95:2c:65:f7:34:
                    2c:67:1c:93:00:81:a7:e2:23:da:1a:3c:d1:9f:84:
                    5e:01:3f:71:e7:9c:cd:e0:4f:fc:db:a6:2f:33:3a:
                    3d:ce:6d:52:72:47:0b:08:9c:04:1f:4a:cd:cd:71:
                    db:c2:3f:0d:9c:b4:24:ca:25:06:49:2b:40:a7:96:
                    b6:60:b7:8d:c7:b0:b4:84:96:06:63:3b:d9:0c:25:
                    8d:af:ad:90:ce:b8:d5:c6:e6:28:28:bd:4b:72:92:
                    28:1a:0a:b7:15:3c:28:26:15:ab:fc:88:22:74:50:
                    77:cc:3d:a3:c8:be:83:14:3d:ca:0e:79:aa:71:66:
                    56:b8:6f:fe:2a:2d:36:ff:0c:af:b9:61:5c:5b:5f:
                    a4:cc:0a:5b:13:31:c9:16:3f:51:9c:19:56:dd:06:
                    1d:c9:6f:f6:17:61:61:7b:4c:cb:aa:b9:92:52:25:
                    9b:8f:02:2d:51:39:5f:f0:89:e2:e8:25:6f:04:2a:
                    d3:6f:a3:3e:a7:44:a8:a1:db:01:55:ad:1d:3f:72:
                    3a:9a:b7:0f:35:a3:de:d2:93:d7:7c:d6:12:66:b2:
                    f9:da:c4:e3:e6:52:6f:55:07:5c:a2:57:0d:7a:ca:
                    20:5a:59:1b:78:ba:cf:e2:1d:b3:33:0a:53:2e:26:
                    9f:39:2f:ec:48:8b:9f:a0:b9:e8:e6:61:9b:89:34:
                    59:02:07:bb:b4:c4:8d:1d:24:72:ea:1e:7c:5f:a9:
                    a3:96:15:e9:4d:7e:4c:94:eb:cb:af:d2:70:83:78:
                    be:36:eb
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier: 
                B8:27:0E:D4:47:BB:27:66:51:3B:E7:F9:8B:9C:48:2E:3D:FD:C8:97
            X509v3 Authority Key Identifier: 
                keyid:A8:4A:6A:63:04:7D:DD:BA:E6:D1:39:B7:A6:45:65:EF:F3:A8:EC:A1

            Authority Information Access: 
                OCSP - URI:http://ocsp.int-x3.letsencrypt.org/
                CA Issuers - URI:http://cert.int-x3.letsencrypt.org/

            X509v3 Subject Alternative Name: 
                DNS:wolfcstech.cn, DNS:wolfcstech.com, DNS:www.wolfcstech.cn, DNS:www.wolfcstech.com
            X509v3 Certificate Policies: 
                Policy: 2.23.140.1.2.1
                Policy: 1.3.6.1.4.1.44947.1.1.1
                  CPS: http://cps.letsencrypt.org
                  User Notice:
                    Explicit Text: This Certificate may only be relied upon by Relying Parties and only in accordance with the Certificate Policy found at https://letsencrypt.org/repository/

    Signature Algorithm: sha256WithRSAEncryption
         46:a1:fb:1c:fe:6e:ef:af:fc:84:e3:7e:20:1d:c8:0c:0b:e4:
         d2:4b:9e:f6:bc:e5:31:59:08:bb:7e:0d:74:3f:e6:de:39:58:
         e2:f4:fa:bf:5c:26:86:96:19:8f:00:13:17:2b:4f:95:c4:bd:
         02:ad:cd:a6:e5:80:21:f5:ee:e6:4d:01:86:07:82:37:5e:39:
         c9:55:40:ed:08:2e:8d:94:b8:86:2f:15:76:10:bd:97:46:06:
         b3:34:80:12:f4:dc:2a:2a:63:80:36:fe:ef:e1:9e:b6:dc:22:
         51:c7:54:46:1a:b2:c5:e8:62:98:90:46:ea:92:8c:fd:d4:dd:
         00:4f:fb:1e:25:24:93:c1:74:15:07:6f:67:d3:be:5b:47:7e:
         18:56:02:01:55:09:fc:bf:7f:ff:27:fc:db:d8:53:55:02:43:
         2e:a0:23:28:01:4d:4d:f9:bc:02:bc:fe:50:c2:67:d7:d4:48:
         23:c2:0b:25:d4:65:e1:8f:3c:75:12:b6:87:b1:17:86:c8:1a:
         26:72:0e:ba:07:92:c4:87:3e:e1:fc:e3:58:ef:a2:23:43:09:
         85:c4:82:00:04:07:49:06:10:bc:fd:20:67:0f:63:f8:ff:bf:
         7f:6f:da:72:77:51:1d:50:34:07:63:e8:68:e3:ef:70:5f:71:
         b4:11:9e:27
```
可以看到主要包含证书格式的版本号；证书的唯一的序列号；签名算法；颁发者的信息；证书的有效期；证书的使用者的信息、身份信息，在这里主要是几个域名；证书使用者的公钥等等。

# Let's Encrypt证书申请
我们通过一个 **Let's Encrypt** 证书申请过程对证书做更多了解。 [Let's Encrypt](https://letsencrypt.org/) 是一个免费、自动化、开放的证书签发机构，目前它已得到了Mozilla，Chrome等的支持，发展十分迅猛。

## 证书申请
Let’s Encrypt 使用ACME协议验证申请人对域名的控制并签发证书。要获取Let’s Encrypt 证书，需使用ACME客户端进行。Let’s Encrypt [官方推荐](https://letsencrypt.org/getting-started/) 使用Certbot这个功能强大，灵活方便的工具，不过也可以使用[其它的ACME客户端](https://letsencrypt.org/docs/client-options/)。

这里我们通过Certbot申请Let’s Encrypt证书。

### Certbot安装
对于在Ubuntu 14.04平台上部署nginx提供Web服务的情况，应该使用 **certbot-auto** 来安装：
```
$ wget https://dl.eff.org/certbot-auto
$ chmod a+x certbot-auto
```

 **certbot-auto** 接收与 **certbot** 完全相同的标记；不带参数执行这个命令时，会自动安装依赖的所有东西，并更新工具本身。执行如下命令：
```
$ ./certbot-auto
```
### 配置Web服务器
 **Let’s Encrypt** 在签发证书之前，需要先通过ACME验证申请者对域名的控制权。验证方法是，ACME客户端产生一些临时文件放在指定的位置，并将该文件的相关信息发送给 **Let’s Encrypt** 。 **Let’s Encrypt**通过http协议访问域名下的对应文件，验证申请者对域名的控制权。因而申请证书前需要先配置Web服务器。

对于nginx服务器而言，可以这样配置：
```
        server {
                listen 80;
                server_name       www.wolfcstech.com wolfcstech.com;
                server_tokens     off;

                access_log        /dev/null;

                if ($request_method !~ ^(GET|HEAD|POST)$ ) {
                        return        444;
                }
               location ^~ /.well-known/ {
                       alias         /home/www-data/www/challenges/;
                       try_files     $uri =404;
               }
                location / {
                        root /home/www-data/www/hanpfei-documents/public;
                        index index.html;
                }
        }
```
### 申请证书
 **Certbot** 功能非常强大，它支持许多的插件，为许多平台上的Web服务器自动地申请、安装及部署证书。对于nginx，目前还不支持证书的自动安装和部署，这里使用 **certonly** 命令单独地获取证书。

执行如下命令：
```
# ../certbot-auto certonly --rsa-key-size 4096 --webroot -w /home/www-data/www/chanllenges/ -d www.wolfcstech.com -d wolfcstech.com
```
主要的参数说明：
 * `--rsa-key-size`：指定RSA密钥，即非对称加密私钥的强度，这里指定为4096位。这个参数用于生成RSA的私钥。
 * `--webroot`：webroot是一个插件，可以与网站的根目录路径一起工作。
 * `-w`：用于指定网站根目录路径。ACME客户端产生的临时文件都将放在这个参数指定的目录下。这个参数的值要与配置的Web服务器的网站根目录路径匹配。
 * `-d`：用于指定要认证的域名。可以用一个证书为多个域名签名。


 **Certbot** 自动地完成证书的申请过程。

![Request Certificate](http://upload-images.jianshu.io/upload_images/1315506-a8c6e0d4ee7eb1c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

证书申请成功之后可以看到如下的提示：

![Request Certificate Successfully](http://upload-images.jianshu.io/upload_images/1315506-8d6c07987333dcb0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

更多关于使用 **Certbot** 申请 **Let’s Encrypt** 证书的信息，可以参考 [**Certbot** 官网](https://certbot.eff.org/)。

从上面的图可以看到证书申请的大体过程：

![Certificate Request Flow](http://upload-images.jianshu.io/upload_images/1315506-2d2891c77738d687.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

申请得到的证书及相关文件被放在`/etc/letsencrypt/`目录下：
```
# find /etc/letsencrypt/
/etc/letsencrypt/
/etc/letsencrypt/accounts
/etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org
/etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory
/etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory/c784b1e1bc605f9cffba9f0888f3e248
/etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory/c784b1e1bc605f9cffba9f0888f3e248/regr.json
/etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory/c784b1e1bc605f9cffba9f0888f3e248/meta.json
/etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory/c784b1e1bc605f9cffba9f0888f3e248/private_key.json
/etc/letsencrypt/archive
/etc/letsencrypt/archive/www.wolfcstech.com
/etc/letsencrypt/archive/www.wolfcstech.com/cert.pem
/etc/letsencrypt/archive/www.wolfcstech.com/chain.pem
/etc/letsencrypt/archive/www.wolfcstech.com/fullchain.pem
/etc/letsencrypt/archive/www.wolfcstech.com/privkey.pem
/etc/letsencrypt/csr
/etc/letsencrypt/csr/0000_csr-certbot.pem
/etc/letsencrypt/keys
/etc/letsencrypt/keys/0001_key-certbot.pem
/etc/letsencrypt/live
/etc/letsencrypt/live/www.wolfcstech.com
/etc/letsencrypt/live/www.wolfcstech.com/cert.pem
/etc/letsencrypt/live/www.wolfcstech.com/chain.pem
/etc/letsencrypt/live/www.wolfcstech.com/privkey.pem
/etc/letsencrypt/live/www.wolfcstech.com/fullchain.pem
/etc/letsencrypt/options-ssl-apache.conf
/etc/letsencrypt/renewal
/etc/letsencrypt/renewal/www.wolfcstech.com.conf
```
 * `/etc/letsencrypt/archive/[域名]`下保存与特定域名相关的文件，包括网站证书、中间证书链、完整证书链和私钥。如果多次为相同的域名申请证书，这个目录下将有多份证书相关文件，文件名后加数字以区分。
 * `/etc/letsencrypt/csr`下是申请证书的`证书签名请求(CSR)`文件。如果多次申请了证书，这个目录下会保存所有申请的文件。
 * `/etc/letsencrypt/live/[域名]`下是最近一次为特定域名申请证书的相关文件，同样是网站证书、中间证书链、完整证书链和私钥。
 * `/etc/letsencrypt/renewal/[域名].conf`文件则保存证书申请的配置信息，以方便下次以相同配置为同样的域名更新证书。

可以看一下证书申请的配置文件内容来对证书申请过程做更多的了解：
```
# cat /etc/letsencrypt/renewal/www.wolfcstech.com.conf
# renew_before_expiry = 30 days
version = 0.9.3
cert = /etc/letsencrypt/live/www.wolfcstech.com/cert.pem
privkey = /etc/letsencrypt/live/www.wolfcstech.com/privkey.pem
chain = /etc/letsencrypt/live/www.wolfcstech.com/chain.pem
fullchain = /etc/letsencrypt/live/www.wolfcstech.com/fullchain.pem

# Options used in the renewal process
[renewalparams]
authenticator = webroot
installer = None
account = c784b1e1bc605f9cffba9f0888f3e248
webroot_path = /home/www-data/www/chanllenges,
rsa_key_size = 4096
[[webroot_map]]
www.wolfcstech.com = /home/www-data/www/chanllenges
wolfcstech.com = /home/www-data/www/chanllenges
```

## CSR文件的内容
证书主要是为网站的公钥签名的，但前面申请证书的过程却并没有看到网站公钥的生成。这是因为公钥是在产生CSR文件的过程中自动生成并保存在CSR文件中的。

使用`openssl`解析前面我们申请**Let’s Encrypt** 证书时产生的CSR文件：
```
# openssl req -in /etc/letsencrypt/csr/0000_csr-certbot.pem -noout -text
Certificate Request:
    Data:
        Version: 2 (0x2)
        Subject: CN=www.wolfcstech.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:b3:0d:f5:cb:a3:9b:94:fb:7e:83:15:72:65:db:
                    3c:56:1d:25:26:b5:5e:88:28:98:0f:c5:d7:df:78:
                    ee:8a:c3:aa:06:5c:0c:81:4d:4f:e6:d9:dd:ed:5d:
                    f2:47:2a:a0:d4:94:a2:18:3c:10:3a:73:0a:aa:24:
                    72:b3:5a:24:70:aa:ff:90:1c:5a:60:cd:f4:de:d6:
                    16:c2:e2:9f:df:d0:b1:ff:28:2d:2d:04:5d:7f:df:
                    aa:9a:11:99:d2:98:82:c1:16:9e:db:c6:d6:99:4f:
                    b8:b6:74:f8:15:47:41:d3:06:cf:10:59:77:f0:f6:
                    71:7d:73:c5:03:6f:d6:3a:fa:a8:bf:d3:c5:44:27:
                    5f:99:91:7f:83:74:b4:94:ee:be:19:da:2d:86:94:
                    1d:7e:7f:e9:5d:a2:15:1e:4d:09:13:4f:06:65:17:
                    95:82:66:5b:39:cb:76:42:87:db:2a:e2:a4:89:88:
                    16:64:d8:af:6a:80:f7:21:50:08:a4:2b:0f:78:36:
                    b3:50:3c:ec:eb:b0:27:5f:d1:89:ee:08:39:d8:71:
                    75:d0:0c:70:6c:c5:94:96:bf:45:cd:4d:8c:66:0d:
                    07:48:78:d5:94:e4:a4:4d:73:1c:7e:60:31:ae:5c:
                    72:4a:e4:11:9b:06:8b:2d:1c:69:54:f0:73:70:d8:
                    17:1c:2c:f9:24:20:e1:33:e0:dd:ec:a6:3c:53:0e:
                    1f:d7:83:24:cd:33:f9:94:e9:e6:3e:8e:76:e7:77:
                    3c:57:78:08:d4:ab:70:35:f6:a0:13:b5:ba:02:bc:
                    88:a9:9c:d5:47:62:99:f1:a4:08:a7:a3:22:79:73:
                    c4:77:2a:49:58:f2:ec:d1:87:13:ed:76:62:23:09:
                    1f:bf:22:e4:80:21:49:a1:43:7e:a6:76:67:30:32:
                    c3:9e:40:8e:a1:8c:d6:09:31:be:d9:7b:b3:73:8a:
                    a9:75:cf:66:2f:1e:a0:e3:01:5b:41:30:fd:68:ae:
                    88:cd:75:fa:72:32:d7:92:fc:a8:5c:eb:2b:82:c8:
                    06:e5:53:08:8d:14:92:ab:d9:81:96:45:16:43:5a:
                    52:12:ba:3c:51:55:c8:90:24:41:95:f7:bd:a0:d1:
                    7d:62:2a:56:30:a6:8e:5e:7c:8b:69:b3:ab:d3:24:
                    c7:35:89:eb:df:4d:c6:a8:0c:74:1d:d9:2b:30:67:
                    2b:ac:3f:a8:1a:c2:76:23:92:1d:00:96:1c:95:aa:
                    da:a6:51:61:30:b2:d0:42:a2:81:51:04:4c:5f:78:
                    e9:3c:6b:e6:1d:22:b2:80:3d:96:6c:2d:43:fd:ed:
                    82:9f:5f:59:f0:f3:44:a8:82:3f:5b:63:e1:4d:cb:
                    84:ce:dd
                Exponent: 65537 (0x10001)
        Attributes:
        Requested Extensions:
            X509v3 Subject Alternative Name: 
                DNS:www.wolfcstech.com, DNS:wolfcstech.com
    Signature Algorithm: sha256WithRSAEncryption
         a5:e8:87:4a:fa:db:fe:b5:10:d5:39:c2:a5:88:79:9a:25:d9:
         f2:3b:e2:ea:46:0d:18:28:b2:0d:87:df:85:9d:0e:4a:82:bd:
         30:1c:74:6d:4c:43:46:33:82:b4:53:3b:a7:22:a5:29:04:92:
         05:50:f9:9b:c7:33:d6:41:0d:5b:9a:bd:d5:d7:98:cc:5c:45:
         13:46:8f:56:29:c7:ba:72:81:71:23:85:33:cb:68:d2:e7:b8:
         08:9f:40:7e:9f:51:62:a9:50:6a:ab:63:de:f5:d5:30:ee:c4:
         6b:40:4a:37:85:fb:51:15:a2:e4:de:58:cf:65:8c:c6:52:23:
         2c:1c:6e:b0:32:bb:20:b8:a5:50:6c:0f:69:32:b2:58:e8:cd:
         d9:11:47:eb:09:f2:d1:31:0c:0c:0a:6b:d9:64:ed:b7:8a:49:
         e0:28:18:dd:3d:94:88:85:4d:bc:be:0b:96:bb:f9:f2:b4:83:
         45:54:78:d2:12:a8:b9:28:f7:42:88:ab:31:74:0b:ea:7d:c9:
         8f:0c:a1:ad:5d:28:b9:6f:da:02:6f:c6:ba:d7:77:22:bb:e4:
         20:74:c6:75:85:63:1b:da:b8:59:50:1c:76:75:cc:c4:93:28:
         cb:e4:c4:4b:dc:40:e6:b7:f5:dc:fd:5c:32:cf:8e:f5:03:9a:
         0b:67:99:48:d2:88:ba:e4:97:fd:8e:17:ae:8f:fb:80:5b:32:
         4c:d4:63:65:37:32:c7:4f:7f:9f:86:67:3e:20:fe:94:d1:b3:
         82:7d:72:db:00:91:40:1a:9a:9b:82:38:9b:44:90:3e:36:4d:
         fa:40:53:fc:18:4d:e1:78:21:b7:31:0e:62:9a:52:55:be:24:
         96:07:2b:53:77:1f:5e:10:62:79:85:57:bc:4c:b7:f5:9b:47:
         d3:00:72:dd:19:22:81:04:d6:77:26:47:2c:56:63:1d:e8:51:
         ec:61:2e:ff:a4:c5:ea:1c:6d:c3:42:bc:bc:38:b8:6d:8d:c9:
         cc:a5:67:35:26:dc:09:a7:c9:e7:0f:ee:82:7b:ac:59:4a:b8:
         ee:75:2b:47:78:51:f4:b9:27:64:a0:af:18:3d:2a:d8:e7:34:
         b7:0d:5e:c1:49:77:25:33:50:80:f8:8d:45:59:fd:18:c2:f4:
         10:f0:7d:81:28:d0:16:c5:a5:3e:0b:53:78:99:19:10:50:95:
         e6:41:4b:49:d6:61:b3:82:48:03:e9:ba:a1:aa:cc:73:f0:08:
         83:44:88:cf:fc:64:03:5e:96:9d:2d:a3:fc:96:50:c9:73:3f:
         3f:5b:92:46:d3:ec:2d:df:d1:a8:9d:87:be:fc:17:22:e2:21:
         1b:2a:14:6a:e3:26:e5:7b
```
可以看到CSR文件包含了加密算法的信息(RSA)，公钥，公钥的大小(4096位)，签名算法 (sha256WithRSAEncryption)，域名等信息。

上面的CSR文件实际通过类似下面的命令生成：
```
$ openssl req -new -sha256 -key /etc/letsencrypt/live/www.wolfcstech.com/privkey.pem -out /etc/letsencrypt/csr/0000_csr-certbot.pem
```

也可以借助于openssl，通过私钥单独生成公钥：
```
# openssl rsa -in /etc/letsencrypt/live/www.wolfcstech.com/privkey.pem -pubout -out rsa_public_key.pem
```

## 证书的选择
证书体系的安全性非常依赖CA的私钥的强度，以及CA的私钥的保密性。如果有财大气粗者，建造了计算能力超强的计算机，计算出了CA的私钥；或者CA的安全系统遭受了攻击，结果私钥被盗；又或者CA内部有图谋不轨者盗走了CA的私钥，则拿到私钥的人就可以随意为各网站的仿冒者签发可以通过安全验证的证书了。

不同CA在维护安全方面的实力的差异而造成了不同证书间安全性的差异。提供安全服务的人，可以根据自己对安全性的需求，选择适当的证书。购买收费的证书似乎主要是投资给CA，以促使其加强自身安全体系的建设，防止私钥出现安全性问题。

当前市场占有率排名前10的CA大概占有90%以上的市场份额。商业的CA主要有如下的这些：

[Network solutions](https://www.networksolutions.com/SSL-certificates/index-res.jsp?bookmarked=0c497854fcbf6d22e7b069ceddb8.076)

[Entrust  SSL Certificates](https://www.entrust.com/ssl-certificates/)

[Symantec SSL Certificates](https://www.symantec.com/ssl-certificates/)

[Digicert – SSL Digital Certificate Authority](https://www.digicert.com/)

[Thawte](https://www.thawte.com/)

[Rapid SSL – SSL Certificate](https://www.rapidssl.com/)

[Comodo – SSL Certificate Authority](https://www.comodo.com/)

[StartCom](https://www.startssl.com/)

[GlobalSign](https://www.globalsign.com/en/)

[GoDaddy SSL Certificates](https://sg.godaddy.com/zh/web-security/ssl-certificate)

此外，还有一些非营利性质的CA，主要有如下这些：

[Let's Encrypt](https://letsencrypt.org/)

[CAcert](http://www.cacert.org/)

# 参考文档：
[HTTPS权威指南](https://book.douban.com/subject/26869219/)

[What is a CSR (Certificate Signing Request)?](https://www.sslshopper.com/what-is-a-csr-certificate-signing-request.html)

[Certificate Decoder](https://www.sslshopper.com/certificate-decoder.html)

[The Most Common OpenSSL Commands](https://www.sslshopper.com/article-most-common-openssl-commands.html)

[What is a Digital Signature?](http://www.youdzone.com/signature.html)

[Description of Symmetric and Asymmetric Encryption](https://support.microsoft.com/en-us/kb/246071)

[Certificate signing request](https://en.wikipedia.org/wiki/Certificate_signing_request)

[Certificate authority](https://en.wikipedia.org/wiki/Certificate_authority)

[How to choose the right Certificate Authority for your Web site](http://www.itbusiness.ca/blog/how-to-choose-the-right-certificate-authority-for-your-web-site/20830)

[10 World Popular Cheap SSL Certificate Providers](http://www.iseenlab.com/cheap-ssl-certificate-providers/)

[10 best SSL certificate providers](https://securitygladiators.com/2016/02/28/best-ssl-certificates-providers/)

[HTTPS 升级指南](http://www.ruanyifeng.com/blog/2016/08/migrate-from-http-to-https.html)

[Let's Encrypt，免费好用的 HTTPS 证书](https://imququ.com/post/letsencrypt-certificate.html)

[数字签名是什么？](http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)

[openssl 生成rsa私钥、公钥和证书](http://www.fzb.me/2015-1-15-openssl-rsa.html)

[数字签名、数字证书、对称加密算法、非对称加密算法、单向加密（散列算法）](http://www.cnblogs.com/JCSU/articles/2803598.html)
