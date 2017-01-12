---
title: Debian Jessie基于PHP7安装Pydio
date: 2016-12-24 01:18:09
tags: [Pydio, PHP, Debian]
thumbnail: /images/covers/install-pydio-on-debian-jessie-with-php7.png
---

> **Edit history**
>
> *2017/01/12*
>
> * 更正一些错误
> * 完善教程
> * 增加MySQL的配置

之前我在树莓派上安装的是ownCloud，然而由于在校园网，外网无法访问，而ownCloud的安装实在有些麻烦，于是我在DigitalOcean新开了一个droplet，打算在上面安装Pydio。

## 安装Nginx

为了防止遇到不支持的参数，我决定直接从官方源安装。参照官方文档，添加key：

````
$ wget http://nginx.org/keys/nginx_signing.key
$ sudo apt-key add nginx_signing.key
````

然后把以下内容放进`/etc/apt/sources.list.d/nginx.list`：

````
deb http://nginx.org/packages/mainline/debian/ jessie ngin
deb-src http://nginx.org/packages/mainline/debian/ jessie nginx
````

保存之后运行：

````
$ sudo apt-get update
$ sudo apt-get install nginx
````

## 从DotDeb安装php7.0

Debian Jessie的官方源并不提供php7.0，所幸我们可以通过DotDeb提供的源安装。DotDeb为Debian维护了几个常用服务器组件的官方最新源，其实他们也提供了Nginx，只不过我是后来才发现的……

添加以下内容到`/etc/apt/sources.list.d/dotdeb.list`：

````
deb http://packages.dotdeb.org jessie all
deb-src http://packages.dotdeb.org jessie all
````

然后同样添加key：

````
$ wget https://www.dotdeb.org/dotdeb.gpg
$ sudo apt-key add dotdeb.gpg
````

最后：

````
$ sudo apt-get update
$ sudo apt-get install php7.0-fpm
````

安装完成之后，我们需要做一些配置。首先是`fix_pathinfo`的修改。DotDeb提供的php7.0的`php.ini`文件位置和默认的是不一样的……通过以下命令可以的出`php.ini`的位置：

````
$ php --ini
Configuration File (php.ini) Path: /etc/php/7.0/cli
Loaded Configuration File:         /etc/php/7.0/cli/php.ini
Scan for additional .ini files in: /etc/php/7.0/cli/conf.d
Additional .ini files parsed:      /etc/php/7.0/cli/conf.d/10-opcache.ini,
/etc/php/7.0/cli/conf.d/10-pdo.ini,
/etc/php/7.0/cli/conf.d/20-calendar.ini,
/etc/php/7.0/cli/conf.d/20-ctype.ini,
/etc/php/7.0/cli/conf.d/20-exif.ini,
/etc/php/7.0/cli/conf.d/20-fileinfo.ini,
/etc/php/7.0/cli/conf.d/20-ftp.ini,
/etc/php/7.0/cli/conf.d/20-gettext.ini,
/etc/php/7.0/cli/conf.d/20-iconv.ini,
/etc/php/7.0/cli/conf.d/20-json.ini,
/etc/php/7.0/cli/conf.d/20-phar.ini,
/etc/php/7.0/cli/conf.d/20-posix.ini,
/etc/php/7.0/cli/conf.d/20-readline.ini,
/etc/php/7.0/cli/conf.d/20-shmop.ini,
/etc/php/7.0/cli/conf.d/20-sockets.ini,
/etc/php/7.0/cli/conf.d/20-sysvmsg.ini,
/etc/php/7.0/cli/conf.d/20-sysvsem.ini,
/etc/php/7.0/cli/conf.d/20-sysvshm.ini,
/etc/php/7.0/cli/conf.d/20-tokenizer.ini
````

~~于是我们修改`/etc/php/7.0/cli/php.ini`，找到`cgi.fix_pathinfo=0`，把`1`改为`0`。~~**由于我们的Pydio是通过fpm运行的，而fpm本身有`php.ini`，所以这里先不配置。**

接着，我们修改fpm的配置，它在`/etc/php/7.0/fpm/pool.d/www.conf`。确定如下两行内容：

````
user = www-data
group = www-data
````

最后别忘了重启php-fpm：

````
$ sudo systemctl restart php7.0-fpm.service
````

## 下载Pydio

这一部分就比较简单了，直接从官方的下载链接wget下来并解压到/`var/www`即可。另外，官方源的Nginx默认的html目录并不在`/var/www`，需要手动创建目录。

````
$ sudo mkdir /var/www
$ wget https://download.pydio.com/pub/core/archives/pydio-core-7.0.3.tar.gz
$ tar -xzf pydio-core-7.0.3.tar.gz
$ sudo mv pydio-core-7.0.3 /var/www/pydio
$ sudo chown -R www-data:www-data /var/www/pydio/data
$ sudo chown www-data:www-data /var/www/pydio/.htaccess
````

注意我们的fpm和Nginx的worker都是以www-data用户运行的（Nginx的worker默认不是www-data运行，但我后面会更改），因此需要保证Pydio的data目录是www-data可读写的。*`.htaccess`文件后面也需要可写，所以这里一并修改了。*

## 为Pydio配置Nginx和php-fpm

由于没有使用apache，php-fpm就成为了处理php请求的服务。注意Pydio的目录下提供了一个`nginx.conf.sample`，但是这个配置不能拿来就用，需要修改如下：

````
server {
        listen 80;
        server_name your-pydio.your-domain.tld;
        keepalive_requests    10;
        keepalive_timeout     60 60;

        client_max_body_size 800M;
        client_body_buffer_size 128k;

	    # Document Root
        root /var/www/pydio;
        
        index index.php index.html index.htm;

        if (!-e $request_filename){
                rewrite ^(.*)$ /index.php break;
        }

        # nginx configuration
        location ~ \.php$ {
                #include       snippets/fastcgi-php.conf;
                fastcgi_pass  unix:/run/php/php7.0-fpm.sock;
                # Below are from /etc/nginx/conf.d/default.conf
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $request_filename;
                include       fastcgi_params;
        }
        
        location ~ ^/data/public/.*$ {
                allow all;
        }

        location ~ ^/(conf|data) {
                deny all;
        }
}
types {
        application/font-woff2                 woff2;
}
````

可以看到主要的改动是php部分，首先他原来include的`snippets`在这个版本的Nginx里是木有的……所以我参照Nginx的默认设置做了改动。同时，通过`/etc/php/7.0/fpm/pool.d/www.conf`可以发现sock的路径也需要修改。*另外，我还对data目录做了访问控制，后面会提到。*

修改完成，重启Nginx的服务之后，我在pydio的目录下新建了一个`php-test.php`：

````php
<?php phpinfo(); ?>
````

然后访问，结果发现一直502。查看Nginx的log发现居然是Permission Denied，谷歌之后得知，运行fpm的进程用户是`www-data`，而其sock文件的权限是0660，悲剧的是Nginx的worker默认是用`nginx`用户运行的，因此无法把脚本传递给fpm……解决方法是修改`/etc/nginx/nginx.conf`：

````
user  www-data;
````

然后重启Nginx：

````
$ sudo systemctl restart nginx.servive
````

然后访问该php文件，你会看到你的php信息网页。

## 安装PHP扩展

我兴冲冲输入了域名，迎接我的是403。

差了log之后我才想起来我忘了装扩展。官方声明需要的扩展有：

>  Basic PHP extensions required are intl, mbstring, gd and dom-xml. For cryptography, we require either open-ssl (recommended) or mcrypt (will be removed from PHP soon).

通过`php -m`可以看到已经安装的模块。接下来我们安装缺少的模块：

````
$ sudo apt-get install php7.0-mbstring php7.0-intl php7.0-gd php7.0-xml
$ sudo systemctl restart php7.0-fpm.service
````

（其实安装完之后还是403,，因为index的原因，我重新修改了Nginx的配置，已经一并写到前面提供的了- -）

## 解决错误

现在输入域名，你会看到Pydio Disgnostic Tool列举出的所有有问题的地方。我遇到的问题是3个：

*  Security Breach，主要是data目录权限配置错误
*  SSL Encryption，由于没有启用HTTPS所以是WARNING
*  PHP Output Buffer disabled，需要将这个设置为禁用以提高性能

这里我遇到了悲剧……之前修改的`/etc/php/7.0/cli/php.ini`是全局的，但是fpm本身还需要一份php.ini的配置，位于`/etc/php/7.0/fpm/php.ini`，**我们先参照之前的内容修改fix_pathinfo**，然后找到并修改如下内容：

````
; Note: Output buffering can also be controlled via Output Buffering Control
;   functions.
; Possible Values:
;   On = Enabled and buffer is unlimited. (Use with caution)
;   Off = Disabled
;   Integer = Enables the buffer and sets its maximum size in bytes.
; Note: This directive is hardcoded to Off for the CLI SAPI
; Default Value: Off
; Development Value: 4096
; Production Value: 4096
; http://php.net/output-buffering
output_buffering = Off
````

注意我把`output_buffering`改为了`Off`。

重启fpm服务之后，刷新网页，解决！

data权限问题源于没有在nginx中屏蔽data目录的访问，*这个问题如果按照我提供的配置，那么将不会遇到。我之前不知道，才遇到这个问题。以下的修改我已经加到前面了。*，所以我们修改`/etc/nginx/conf.d/pydio.conf`，在`server`中增加如下内容（参考自[http://martin-denizet.com/nginx-configuration-for-pydio-with-ssl/](http://martin-denizet.com/nginx-configuration-for-pydio-with-ssl/)）：

````
server {
	# ...

	location ~ ^/data/public/.*$ {
		allow all;
	}

	location ~ ^/(conf|data) {
		deny all;
	}
}
````

上述内容会屏蔽除了`/data/public/*`以外的所有data内容。重启Nginx服务之后刷新页面，只剩下一个SSL的warning了，这个我们下次再说Orz

## 配置SQLite

**SQLite容易引发配置文件被锁定问题，因此建议使用MySQL（见下面）。**

开始安装向导之后，我才发现Pydio的配置数据（不包含用户文件）需要存储到数据库里，在Pydio支持的数据库中，由于MySQL和PostgreSQL都太大了，于是我决定使用SQLite 3。

安装php扩展：

````
$ sudo apt-get install php7.0-sqlite3
$ sudo systemctl restart php7.0-fpm.service
````

后面需要的缓存等我都暂时没启用。

## 配置MySQL

如果你想使用MySQL，参照官网，可以使用如下命令安装：

````
$ wget https://dev.mysql.com/get/mysql-apt-config_0.8.1-1_all.deb
$ sudo dpkg -i mysql-apt-config_0.8.1-1_all.deb
````

你可以选择需要安装的版本以及是否需要其他工具（Workbench等）。

然后就可以安装MySQL服务器和php的相关扩展了：

````
$ sudo apt-get install mysql-server php7.0-mysql
$ sudo systemctl restart php7.0-fpm
````

中间需要设置root帐号的密码。安装完成之后，我们需要为Pydio创建数据库：

````
$ mysql -p
...
mysql> CREATE DATABASE pydio;
Query OK, 1 row affected (0.00 sec)

mysql> USE pydio;
Database changed
mysql> CREATE USER 'pydio'@'localhost' IDENTIFIED BY 'password';
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT ALL PRIVILEGES ON pydio.* TO 'pydio'@'localhost';
Query OK, 0 rows affected (0.00 sec)

mysql> exit
````

上面的命令会创建名为pydio的数据库，然后添加一个可以管理这个数据库的用户，用户名pydio密码password。接下来，只要在Pydio中选择MySQL并输入数据库名字、用户名、密码即可。

## 修改htaccess文件

*这里的配置也在前面已经解决了。*安装过程中，他提示我无法写入该文件，直接改用户！

````
$ sudo chown www-data:www-data .htaccess
````

刷新页面之后，就可以正常使用了！

## 扩大上传限制

Pydio默认的上传大小限制在2M，显然是不够的，参考[Topic: [solved] How to change Upload limit | Pydio](https://pydio.com/forum/f/topic/solved-how-to-change-upload-limit/)，可以改变上传大小限制：

首先编辑`/etc/php/7.0/fpm/php.ini`，修改两个地方：

````
upload_max_filesize=1024M
post_max_size=1280M
````

以上两个配置分别是上传文件的上限和POST数据的上限，所以后者比前者需要大一些。你还可以修改`max_file_uploads`来设置同时上传的文件数量上限。

重启fpm服务，然后进入到`Settings - Application Parameters - Application Core - Uploaders Options`，设置`Limitations - File Size`为`1024M`，然后点击右上角保存即可。