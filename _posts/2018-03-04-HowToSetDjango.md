---
layout: post
author: OSer Huang
title: 如何在Ubuntu16.04上通过Nginx和Gunicorn部署Django（译）
comments: true
---
<!-- # 如何在Ubuntu16.04上通过Nginx和Gunicorn部署Django（译） -->

原文地址：[https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04)

![HowToSetUpDjango][1]

### 简介

Django是一个强大的web框架，它可以帮助你构建Python应用或网站。Django包含了一个简单的server，你可以用它在本地测试代码。但是在现实的生产环境中， 我们需要一个更安全更强大的web server。
在这份教程中，我们将演示如何在Ubuntu 16.04中安装和配置一些组件来支持和服务Django应用。我们先创建一个PostgreSQL数据库来替代默认的SQLite数据库。然后我们会配置Gunicorn与我们的应用进行交互。最后我们创建Nginx作为Gunicorn的反向代理（reverse proxy），我们将使用其安全和性能特性来为我们的应用服务。

## 前提与目标

要完成这个教程，你需要一个全新的Ubuntu 16.04实例，并配置一个有<code>sudo</code>权限的非root用户。你可以通过我们的[**服务器初始化教程**](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)来学习如何操作。
我们会在虚拟环境中安装Django。将Django安装到你的项目工程环境中可以使你分开操作你的项目和其所需的工程环境。
当我们创建并运行了数据库和我们的应用后，我们会安装配置Gunicorn应用服务器。 它将充当我们应用的一个接口，将客户端的HTTP请求转换为我们应用可以处理的Python调用。然后我们会为Gunicorn配置Nginx，我们会用到其高性能连接处理机制和一些能简单部署的安全特性。

## 从Ubuntu代码库安装程序包

首先，我们要从Ubuntu代码库下载并安装所有需要的项目，之后我们会使用Python包管理器<code>pip</code>来安装额外的组件。
我们需要先更新本地的<code>apt</code>包目录，然后下载安装软件包。根据你使用的Python版本的不同，你所需要安装的包也不同。
如果你在使用**Python 2**，输入：

```console
$ sudo apt-get update
$ sudo apt-get install python-pip python-dev libpq-dev postgresql postgresql-contrib nginx
```

如果你在使用**Python 3**，输入：

```console
$ sudo apt-get update
$ sudo apt-get install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx
```

这样我们就安装了<code>pip</code>，之后构建Gunicorn所需的一些Python开发文件，Postgres数据库及一些它所需要的库，和Nginx网页服务器。

## 创建PostgreSQL数据库和用户

现在，我们要为Django应用创建一个数据库和数据库用户。
默认配置下，Postgre对本地连接使用“同行认证（peer authentication）”的认证机制。简单来说就是如果一个操作系统的用户名和一个合法的Postgres用户名相同，那么这个用户就可以直接登录不需要其他授权。
在安装Postgres时，会自动创建一个名为<code>postgres</code>操作系统用户用以匹配PostgreSQL的管理员用户<code>postgres</code>。我们需要用这个用户来进行一些管理员操作。我们可以使用<code>sudo</code>并用<code>-u</code>选项传递用户名。
输入以下命令进入Postgres交互会话:

```console
$ sudo -u postgres psql
```

你将会看到PostgreSQL的提示符，在这个提示符下可以根据我们的要求进行设置。  
首先，我们为我们的项目创建一个数据库： 

```sql
posegres=# CREATE DATABASE myproject;
```

>***注意**：每条Postgres语句必须以分号结尾，当你遇到问题时请检查你的命令是否有分号。*

然后为我们的项目创建一个数据库用户。请确保你选择了一个安全的密码：

```sql
posegres=# CREATE USER myprojectuser WITH PASSWORD 'password';
```

接下来我们更改这个新创建用户的一些连接参数。这是为了加快数据库操作的速度，我们就不需要在每次建立连接时都查询并设置值。  
我们将默认编码方式设置为UTF-8，这也是Django的默认编码方式。同时，我们将默认的事务隔离机制（transaction isolation scheme）设为“read committed”，这将禁止未提交事务的读取操作。最后，我们修改时区。默认设置下，我们的Django项目会设为<code>UTC</code>。这都是Django的[**推荐设置**](https://docs.djangoproject.com/en/1.9/ref/databases/#optimizing-postgresql-s-configuration)：

```sql
posegres=# ALTER ROLE myprojectuser SET client_encoding TO 'utf8';
posegres=# ALTER ROLE myprojectuser SET default_transaction_isolation TO 'read committed';
posegres=# ALTER ROLE myprojectuser SET timezone TO 'UTC';
```

现在，为新用户授予数据库的管理权限：

```sql
posegres=# GRANT ALL PRIVILEGES ON DATABASE myproject TO myprojectuser;
```

但你完成了所有操作后，你可以输入以下命令退出PostgreSQL提示符：

```sql
posegres=# \q
```

## 为项目创建Python虚拟环境

现在我们创建了数据库，我们继续完善项目的其他需求。为了能更简单的管理，接下来我们将创建Python虚拟环境。
首先，我们使用<code>pip</code>安装<code>virtualenv</code>。
如果你在使用**Python 2**，输入以下命令更新<code>pip</code>和安装软件包：

```console
$ sudo -H pip install --upgrade pip
$ sudo -H pip install virtualenv
```

如果你在使用**Python 3**，输入以下命令更新<code>pip</code>和安装软件包：

```console
$ sudo -H pip3 install --upgrade pip
$ sudo -H pip3 install virtualenv
```

安装<code>virtualenv</code>之后，我们就可以开始创建我们的工程项目。为我们的工程文件创建一个目录，然后进入该目录：

```console
$ mkdir ~/myproject
$ cd ~/myproject
```

在工程目录下创建Python虚拟环境：

```console
$ virtualenv myprojectenv
```

这将在<code>myproject</code>目录下创建一个名为<code>myprojectenv</code>的文件夹。在该文件夹下会安装本地的Python和<code>pip</code>。我们可以使用它们为我们的项目安装和配置一个Python环境。
在安装其他Python库之前，我们先要激活虚拟环境：

```console
$ source myprojectenv/bin/activate
```

现在命令行提示符会改变以显示你正在一个Python虚拟环境中操作。它应该看起来像这样：<code>(myprojectenv)user@host:~/myproject$</code>。
激活了虚拟环境后，我们就可以用本地的<code>pip</code>安装Django，Gunicorn和PostgreSQL适配器<code>psycopg2</code>:
>***注意**：当在使用虚拟环境时，无论使用的是哪一版本的Python，只需用使用<code>pip</code>命令（不是<code>pip3</code>）。*

```console
(myprojectenv) $ pip install django gunicorn psycopg2
```

现在你已经安装了所有Django项目所需的软件。

## 创建和配置一个新的Django工程

安装完Python组件后，我们现在可以创建Django项目文件。

### 创建Django工程

我们之前已经创建了一个工程目录，我们让Django将文件安装在该目录下。它将创建一个次级目录，里面包含实际的代码和管理的脚本。我们这样做的目的是为了显式地指定目录，而不是让Django来决定：

```console
(myprojectenv) $ django-admin.py startproject myproject ~/myproject
```

现在，你的工程目录（在我们的示例中即~/myproject）下应该有以下内容：

- ~/myproject/manage.py：Django的管理脚本。
- ~/myproject/myproject/：Django工程包，其中应该包含\_\_init\_\_.py，settings.py，urls.py和wsgi.py。
- ~/myproject/myprojectenv/：我们之前创建的虚拟环境目录。

### 配置工程设置

首先，你需要对新建的项目进行设置。在文件编辑器中打开设置文件：

```console
(myprojectenv) $ nano ~/myproject/myproject/settings.py
```

找到`ALLOWED_HOSTS`指令。该指令定义了一个列表，列表中包含可能连接Django实例的服务器地址及域名。任何**Host**头不在列表中的请求都会触发异常。Django要求你这样设置以屏蔽一些安全漏洞。
在方括号中列出与你Django服务器相关联的IP地址和域名。每一项都需要用引号包围并用逗号进行区分。如果你希望一个域名和其所有子域名都能连接，你可以在域名前添加一个点号（如“.example.com”）。在下面的文件片段中，我们用一些例子进行演示：

```python
~/myproject/myproject/settings.py

. . .
# 最简单的例子：只添加Django服务器的域名及IP地址
# ALLOWED_HOSTS = [ 'example.com', '203.0.113.5']
# 在域名前添加点号就可以响应“example.com”及其子域名
# ALLOWED_HOSTS = ['.example.com', '203.0.113.5']
ALLOWED_HOSTS = ['your_server_domain_or_IP', 'second_domain_or_IP', . . .]
```

然后找到数据库接入的设置，它会以`DATABASE`为首。默认的设置是用于SQLite数据库的，我们已经建立了一个PostgreSQL数据库，所以我们需要稍加修改。
用你的PostgreSQL数据库信息修改设置。我们告诉Django使用`psycopg2`适配库。我们还需要告诉它数据库的名字，数据库的用户名及其密码，并且我们要说明这个数据库是存储于本地的。你可以将`Port`设为空字符串：

```python
~/myproject/myproject/settings.py

. . .

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'myproject',
        'USER': 'myprojectuser',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '',
    }
}

. . .
```

然后我们移动到文件底部，添加静态文件的存储地址。Nginx需要这个地址来处理静态文件的请求。添加的这一行告诉Django将静态文件防止于项目根目录下的`static`文件夹中：

```python
~/myproject/myproject/settings.py

. . .

STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```

完成后保存并关闭文件。

### 完成工程初始化

现在我们使用Django的管理脚本将初始数据库架构迁移到创建的PostgreSQL数据库：

```console
(myprojectenv) $ nano ~/myproject/manage.py makemigrations
(myprojectenv) $ nano ~/myproject/manage.py migrate
```

输入以下命令为工程创建一个管理用户：

```console
(myprojectenv) $ ~/myproject/manage.py createsuperuser
```

根据提示信息，你需要创建用户名，提供邮件地址，并设置密码。

将所有静态文件集中到之前配置的静态文件目录：

```console
(myprojectenv) $ ~/myproject/manage.py collectstatic
```

这个操作需要再次确认。确认后所有的静态文件会被放置在你的工程目录下名为<code>static</code>的文件夹中。  

如果你参照了初始化服务器的教程，现在就会有UFW防火墙保护你的服务器。为了能方便测试服务器部署，我们允许网页服务端口号的接入：

```console
(myprojectenv) $ sudo ufw allow 8000
```

最后，我们就可以运行部署的Django服务器并测试其效果：

```console
(myprojectenv) $ ~/myproject/manage.py runserver 0.0.0.0:8000
```

在浏览器中输入服务器的域名或IP地址，在后面加上端口号<code>:8000</code>：

```
http://server_domain_or_IP:8000
```

你应该能看到Django默认的索引页：
![django_index][2]  
在网页URL后面添加<code>/admim</code>，你将会转到管理员登录界面。你可以输入之前<code>/createsuperuser</code>命令所设置的用户名和密码来登录：
![admin_login][3]  
登录之后，你就能看到Django默认的管理界面了：
![admin_interface][4]  
等你玩够了之后，在终端中输入**CTRL-C**关闭服务器。

### 测试Gunicorn伺服功能
在离开虚拟环境之前，我们最后还要测试Gunicorn确保其能服务我们的应用。首先我们进入工程目录下，在用<code>gunicorn</code>命令加载项目的WSGI模块：

```console
(myprojectenv) $ cd ~/myproject
(myprojectenv) $ gunicorn --bind 0.0.0.0:8000 myproject.wsgi
```

这样我们就在和之前运行Django服务器一样的接口上开启了Gunicorn，你可以返回上一步再测试一遍。

>***注意**：由于Gunicorn现在并不知道静态CSS文件的响应地址，管理界面不会应用任何样式。*

我们为Gunicorn指明Django的<code>wsgi.py</code>文件的相对路径，这个文件是整个应用的入口。在这个文件中定义了<code>application</code>函数用来和我们的应用进行通信。想要更多地了解WSGI规范，戳[**这里**](https://www.digitalocean.com/community/tutorials/how-to-set-up-uwsgi-and-nginx-to-serve-python-apps-on-ubuntu-14-04#definitions-and-concepts)。

结束测试后在终端猛戳**CTRL-C**停止Gunicorn。

这样就完成了Django应用的配置，现在我们可以退出虚拟环境：

```console
(myprojectenv) $ deactivate
```

输入后虚拟环境的提示符会被移除。

## 为Gunicorn创建systemd文件

我们虽然已经验证了Gunicorn可以和我们的Django应用互动，但我们应该实现一个更健（zhuang）壮（bi）的方式来开启和停止我们的应用服务器。所以我们需要创建一个systemd文件。

创建systemd文件需要<code>sudo</code>权限：

```console
$ sudo nano /etc/systemd/system/gunicorn.service
```

我们先添加<code>[Unit]</code>部分，这里用于指明元数据（metadata）和依赖（dependencies）。我们放置在这里的说明告诉系统初始化时，只有在获得网络目标后在开始我们的服务：

```
/etc/systemd/system/gunicorn.service

[Unit]
Description=gunicorn daemon
After=network.target
```

接下来添加<code>[Service]</code>部分。首先指明运行服务的用户和组。这里我们使用普通用户，因为它拥有所有相关的文件。为了能让Nginx轻松地和Gunicorn通信，我们使用<code>www-data</code>组。

接下来我们指明工作目录并添加开启服务的命令。在这个例子中，我们必须要指明安装在虚拟环境下的Gunicorn可执行文件的完整路径。由于Nginx是安装在同一台电脑上，我们将这个路径绑定到一个位于项目目录下的Unix套接字。这样比使用网路端口更加安全也更快一些。我们还可以在这里指明其他的可选的设置，比如在下面的例子里，我们指定3个工作进程：

```
/etc/systemd/system/gunicorn.service

[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=sammy
Group=www-data
WorkingDirectory=/home/sammy/myproject
ExecStart=/home/sammy/myproject/myprojectenv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/sammy/myproject/myproject.sock myproject.wsgi:application
```

最后添加<code>[Install]</code>部分。这部分告诉了systemd，如果我们允许服务在boot的时候启动，我们的服务会在标准multi-user系统运行时启动。

```
/etc/systemd/system/gunicorn.service

[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=sammy
Group=www-data
WorkingDirectory=/home/sammy/myproject
ExecStart=/home/sammy/myproject/myprojectenv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/sammy/myproject/myproject.sock myproject.wsgi:application

[Install]
WantedBy=multi-user.target
```

到这里我们的systemd文件就完成了，保存并关闭它。

现在我们可以开启Gunicorn服务并允许其在boot时启动：

```console
$ sudo systemctl start gunicorn
$ sudo systemctl enable gunicorn
```

我们可以通过查看套接字文件判断操作是否成功。

## 检查Gunicorn套接字文件

键入一下命令查看进程状态：

```console
$ sudo systemctl status gunicorn
```

然后在你的工作目录下查看是否存在<code>myproject.sock</code>文件：

```console
$ ls /home/sammy/myproject
```

```console
输出
manage.py  myproject  myprojectenv  myproject.sock  static
```

如果<code>systemctl status</code>显示有错误或者找不到<code>myproject.sock</code>文件，那么Gunicorn一定没有正确地运行。那么我们查看Gunicorn进程的日志：

```console
$ sudo journalctl -u gunicorn
```

我们可以通过查看日志里的消息找出哪里出了问题。很多原因都会导致运行失败，一般是以下这些原因导致Gunicorn不能创建套接字文件：

- 工程文件被<code>root</code>用户所用，而不是一个<code>sudo</code>用户
- <code>/etc/systemd/system/gunicorn.service</code>中的<code>WorkingDirectory</code>路径没有指向工程目录
- <code>gunicorn</code>进程的<code>ExecStart</code>指令配置有误，检查以下几项：
    - <code>gunicorn</code>二进制文件路径指向虚拟环境中二进制文件的实际位置
    - <code>--bind</code>指令指定创建文件的目录Gunicorn可以访问
    - <code>myproject.wsgi:application</code>确实指向WSGI可调用文件。这意味着当你在<code>WorkingDirectory</code>目录下，通过查找<code>myproject.wsgi</code>模块（就是<code>./myproject/wsgi.py</code>文件），你应该能找到名为<code>application</code>的可调用文件。

如果你修改了<code>/etc/systemd/system/gunicorn.service</code>文件，再次读取服务说明以重新载入守护程序并重启Gunicorn进程：

```console
$ sudo systemctl daemon-reload
$ sudo systemctl restart gunicorn
```

请务必解决了所有问题后再进入下一步。

## 配置Nginx为Gunicorn提供代理

完成Gunicorn配置之后，我们需要将让Nginx将流量导向该进程。

首先我们要在Nginx的<code>sites-available</code>目录下创建一个新的server块:

```console
$ sudo nano /etc/nginx/sites-available/myproject
```

在打开的文件中我们添加新的server块。我首先指明该server块监听端口80，这个端口会被用于响应服务器的域名或IP地址：

```
/etc/nginx/sites-available/myproject

server {
    listen 80;
    server_name server_domain_or_IP;
}
```

然后，我们让Nginx忽略寻找网站图标（favicon）带来的错误。我们还将告诉Nginx如何找到放置在<code>~/myproject/static</code>目录下的静态文件。所有的这些文件都有一个标准的URI外加一个“/static”前缀，我们可以添加一个location块来响应其请求：

```
/etc/nginx/sites-available/myproject

server {
    listen 80;
    server_name server_domain_or_IP;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/sammy/myproject;
    }
}
```

最后我们创建一个<code>location / {}</code>块来匹配其他请求。在这个location中，我们导入标准的<code>proxy_params</code>文件。然后我们将流量导入Gunicorn创建的的套接字：

```
/etc/nginx/sites-available/myproject

server {
    listen 80;
    server_name server_domain_or_IP;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/sammy/myproject;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/sammy/myproject/myproject.sock;
    }
}
```

保存并关闭文件。现在我们将其连接至<code>sites-enabled</code>目录使其起作用：

```console
$ sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```

检查Nginx配置文件的语法错误：

```console
$ sudo nginx -t
```

如果没有找到错误就可以重启Nginx：

```console
$ sudo systemctl restart nginx
```

最后我们需要让防火墙允许80端口的访问，并且我们也不在需要8000端口：

```console
$ sudo ufw delete allow 8000
$ sudo ufw allow 'Nginx Full'
```

现在你可以通过域名和IP地址访问你的网站。

原文地址：[https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16:-04)


[1]: /images/blog/2018-03-04-HowToSetDjango/How_To_Set_Up_Django_with_Postgres_Nginx_and_Gunicorn_on_Ubuntu_16.04.png
[2]: /images/blog/2018-03-04-HowToSetDjango/django_index.png
[3]: /images/blog/2018-03-04-HowToSetDjango/admin_login.png
[4]: /images/blog/2018-03-04-HowToSetDjango/admin_interface.png
