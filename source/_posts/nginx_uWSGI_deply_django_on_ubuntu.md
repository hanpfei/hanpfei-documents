---
title: 基于nginx和uWSGI在Ubuntu上部署Django
date: 2016-11-14 11:45:49
categories: 服务器
---

# 1. nginx

启动、停止和重启

```
$ nginx
$ nginx -s stop
$ nginx -s reload
```

<!--more-->

# 2. uWSGI安装
用python的pip安装最简单：
```
apt-get install python-dev #不安装这个，下面的安装可能会失败
pip install uwsgi
```

# 3. 基于uWSGI和nginx部署Django

## 1. 原理

```
Web Client <===> Web Server(nginx) <===> The Socket <===> uWSGI <===> Django
```

## 2.基本测试

### 测试uWSGI是否正常

在django项目的根目录下创建test.py文件，添加源码如下：

```
# test.py
def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return ["Hello World"] # python2
    #return [b"Hello World"] # python3
```

Django项目的创建方法，可参考[Django教程](https://my.oschina.net/wolfcs/blog/364715)。

然后，运行uWSGI:

```
$ uwsgi --http :8000 --wsgi-file test.py
```

参数含义:
 * `http :8000`: 使用http协议，8000端口
 * `wsgi-file test.py`: 加载指定文件 test.py

用浏览器打开下面url，应该显示`hello world`字符串：
```
http://[example.com]:8000
```
如果部署服务器的主机和运行浏览器的主机不是同一台，`[example.com]`要替换为部署服务器的主机实际的域名，若是同一台，`[example.com]`可以替换为localhost，如：
```
http://localhost:8000
```

如果显示正确，说明下面3个环节是通畅的：

```
Web Client <===> uWSGI <===> Python
```

### 测试Django项目是否正常

首先确保 项目 本身是正常的：
```
$ python manage.py runserver 0.0.0.0:8000
```

如果没问题，使用uWSGI把Project拉起来：
```
$ uwsgi --http :8000 --module mysite.wsgi
```

 * `module mysite.wsgi`: 加载wsgi module。`mysite.wsgi`指向一个python模块，即`mysite`目录下名为`wsgi.py`的文件。新建出来的Django项目，它的这个文件的内容大体如下：

```
"""
WSGI config for mysite project.

It exposes the WSGI callable as a module-level variable named ``application``.

For more information on this file, see
https://docs.djangoproject.com/en/1.10/howto/deployment/wsgi/
"""

import os

from django.core.wsgi import get_wsgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")

application = get_wsgi_application()
```

如果Project能够被正常拉起，说明以下环节是通的：

```
Web Client <===> uWSGI <===> Django
```

## 3.配置nginx

安装nginx完成后，如果能正常打开http://hostname，说明下面环节是通畅的：

```
Web Client <===> Web Server(nginx)
```

### 增加nginx配置

* 将`uwsgi_params`文件拷贝到项目文件夹下。`uwsgi_params`文件在/etc/nginx/目录下，也可以从[这个页面](https://github.com/nginx/nginx/blob/master/conf/uwsgi_params)下载

* 在项目文件夹下创建文件mysite_nginx.conf，填入并修改下面内容：

```
 mysite_nginx.conf

# the upstream component nginx needs to connect to
upstream django {
    # server unix:///home/www-data/www/mysite/mysite.sock; # for a file socket
    server 127.0.0.1:8001; # for a web port socket (we'll use this first)
}
# configuration of the server
server {
    # the port your site will be served on
    listen      8000;
    # the domain name it will serve for
    server_name www.wolfcstech.com wolfcstech.com www.wolfcstech.cn wolfcstech.cn; # substitute your machine's IP address or FQDN
    charset     utf-8;

    # max upload size
    client_max_body_size 75M;   # adjust to taste

    # Django media
    location /media  {
        alias /home/www-data/www/mysite_data/media;  # your Django project's media files - amend as required
    }

    location /static {
        alias /home/www-data/www/mysite/static; # your Django project's static files - amend as required
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        include     /home/www-data/www/mysite/uwsgi_params; # the uwsgi_params file you installed
    }
}
```

这个配置文件的写法就像常规的nginx配置文件`/usr/local/nginx/conf/nginx.conf`一样。这个配置文件告诉nginx，请求的资源的路径模式满足`/media/*`和`/static/*`的，要在文件系统中找。

在`/etc/nginx/sites-enabled`目录下创建该文件的符号连接，使nginx能够使用它：

```
sudo ln -s /home/www-data/www/mysite/mysite_nginx.conf /etc/nginx/sites-enabled/
```

### 部署static文件

在Django项目的`mysite/settings.py`文件中文件末尾，添加下面一行内容：

```
STATIC_ROOT = os.path.join(BASE_DIR, "static/")
```

然后运行：

```
$ python manage.py collectstatic
```

Django框架在创建项目时，默认提供了admin等接口，这会将Django框架中这些接口用到的一些静态文件，js，css等文件，拷贝到项目的static目录下，如：

```
root@iZuf6gttapr078fb1vrh8uZ:/home/www-data/www/mysite# python manage.py collectstatic

You have requested to collect static files at the destination
location as specified in your settings:

    /home/www-data/www/mysite/static

This will overwrite existing files!
Are you sure you want to do this?

Type 'yes' to continue, or 'no' to cancel: yes
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/collapse.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/timeparse.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/popup_response.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/SelectFilter2.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/jquery.init.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/collapse.min.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/inlines.min.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/calendar.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/SelectBox.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/urlify.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/prepopulate_init.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/change_form.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/cancel.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/prepopulate.min.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/actions.min.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/core.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/prepopulate.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/actions.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/inlines.js'
Copying '/usr/local/lib/python2.7/dist-packages/django/contrib/admin/static/admin/js/vendor/jquery/LICENSE-JQUERY.txt'
......
```

### 测试nginx

首先重启nginx服务：

```
$ nginx -s stop
$ nginx
```

然后检查media文件是否已经可以正常访问：

在目录/path/to/your/project/project/media directory下添加文件meida.png，然后访问http://example.com:8000/media/media.png ，成功后进行下一步测试。

## 4.nginx、uWSGI和test.py

执行下面代码测试能否让nginx在页面上显示`hello, world`

```
uwsgi --socket :8001 --wsgi-file test.py
```

访问http://example.com:8000，如果显示`hello world`，则说明下面环节是通畅的:

```
Web Client <===> Web Server(nginx) <===> The Socket <===> uWSGI <===> Python
```

### 用UNIX socket取代TCP port

对mysite_nginx.conf做如下修改：

```
server unix:///home/www-data/www/mysite/mysite.sock; # for a file socket
# server 127.0.0.1:8001; # for a web port socket (we'll use this first)
```
重启nginx，并再次运行uWSGI：

```
uwsgi --socket mysite.sock --wsgi-file test.py
```

打开 http://example.com:8000/ ，看看是否成功。

### 如果没有成功

检查 nginx error log (/var/log/nginx/error.log)。如果错误如下：

```
connect() to unix:///home/www-data/www/mysite/mysite.sock failed (13: Permission denied)
```
访问Django应用的接口，报出502 Bad Gateway错误：

![Bad Gateway](https://www.wolfcstech.com/images/badgateway.png)

则添加socket权限再次运行：

```
uwsgi --socket mysite.sock --wsgi-file test.py --chmod-socket=666 # (very permissive)
```
or
```
uwsgi --socket mysite.sock --wsgi-file test.py --chmod-socket=664 # (more sensible)
```

## 5. 基于uWSGI和nginx运行Django应用

如果上面一切都显示正常，则下面命令可以拉起D
jango应用：

```
uwsgi --socket mysite.sock --module mysite.wsgi --chmod-socket=666
```
这里的权限同样要注意，就如同上面测试的那样。

### 用.ini文件配置uWSGI

每次都运行上面命令拉起Django应用实在麻烦，使用.ini文件能简化工作，方法如下：

在应用目录下创建文件`mysite_uwsgi.ini`，填入并修改下面内容：

```
# mysite_uwsgi.ini file
[uwsgi]

# Django-related settings
# the base directory (full path)
chdir           = /home/www-data/www/mysite
# Django's wsgi file
module          = mysite.wsgi
# the virtualenv (full path)
# home            = /path/to/virtualenv

# process-related settings
# master
master          = true
# maximum number of worker processes
processes       = 10
# the socket (use the full path to be safe
socket          = /home/www-data/www/mysite/mysite.sock
# ... with appropriate permissions - may be needed
chmod-socket    = 666
# clear environment on exit
vacuum          = true
```
现在，只要执行以下命令，就能够拉起django应用：

```
uwsgi --ini mysite_uwsgi.ini # the --ini option is used to specify a file
```

### 检查配置

最后，通过访问Django提供的接口接口服务，检查上面的所有配置，如在浏览器中输入`http://www.wolfcstech.com:8000/admin`，应该可以看到如下的页面：

![Django admin page](https://www.wolfcstech.com/images/djang_admin.png)

### uWSGI后台运行
前面介绍的uWSGI运行方式是在终端前台运行，终端关闭的话，uWSGI进程也都会跟着结束掉。可我们也不能总是为uWSGI开一个终端啊。我们可以配置uWSGI后台运行。需要在mysite_uwsgi.ini配置文件中添加：

```
daemonize = /home/www-data/www/web_application_log/uwsgi.log
```
这样就会把日志打印到uwsgi.log中。

### 使uWSGI在系统启动时启动

编辑文件/etc/rc.local, 添加下面内容到这行代码之前exit 0:

```
/usr/local/bin/uwsgi --socket /home/www-data/www/mysite/mysite.sock --module /home/www-data/www/mysite/mysite.wsgi --chmod-socket=666
```

### uWSGI的更多配置

```
env = DJANGO_SETTINGS_MODULE=mysite.settings # set an environment variable
pidfile = /tmp/project-master.pid # create a pidfile
harakiri = 20 # respawn processes taking more than 20 seconds
limit-as = 128 # limit the project to 128 MB
max-requests = 5000 # respawn processes after serving 5000 requests
daemonize = /var/log/uwsgi/yourproject.log # background the process & log
```

## 7. Django文件上传接口

这部分展示一个示例的Django接口。该接口主要用于验证前面的配置。而接口的功能则主要是处理文件上传。

### 配置url模式

在mysite/urls.py中为文件上传接口配置url模式：

```
from django.conf.urls import url
from django.contrib import admin

import uploader

urlpatterns = [
    url(r'^upload', uploader.upload),
    url(r'^admin/', admin.site.urls),
]
```

### 实现上传接口

文件上传接口的实现如：

```
#!/usr/bin/python

from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from django import forms

import hashlib
import json
import os

class NormalUserForm(forms.Form):
  username = forms.CharField()
  headImg = forms.FileField()

dir_path = "/home/www-data/www/mysite_data/uploaded_files"

def cal_md5(diff_file_name):
    md5 = hashlib.md5()
    diff_file = open(diff_file_name, 'rb')
    while True:
        data = diff_file.read(8192)
        if not data:
            break

        md5.update(data)
    md5_value = md5.hexdigest()
    return md5_value

def handle_uploaded_file(file):
    filename = os.path.join(dir_path, file.name)
    print "filename = " + filename
    with open(filename, 'wb+') as destination:
        for chunk in file.chunks():
            destination.write(chunk)
    return cal_md5(filename)

@csrf_exempt
def upload(request):
    md5 = ""
    if request.method == 'POST':
        try:
            files = request.FILES.getlist('file')
            for file in files:
                md5 += handle_uploaded_file(file)
        except Exception,e:
            print str(e)

        if md5 == "":
            hash_md5 = hashlib.md5(request.body)
            md5 = hash_md5.hexdigest()

        data1 = {'md5':md5}
        md5 = json.dumps(data1,sort_keys=True,indent=4)
        print "md5 = " + str(md5)

    return HttpResponse(md5)
```
这个接口将文件保存在由dir_path定义的文件夹下，同时计算文件的md5，返回给客户端。

# 4. 参考文档

[基于nginx和uWSGI在Ubuntu上部署Django](http://www.jianshu.com/p/e6ff4a28ab5a)

[uwsgi后台运行](https://segmentfault.com/q/1010000002523354)

[Python/WSGI 应用快速入门](http://uwsgi-docs-cn.readthedocs.io/zh_CN/latest/WSGIquickstart.html)

[The uWSGI project](http://uwsgi-docs-cn.readthedocs.io/zh_CN/latest/index.html)
