---
title: HTTPS快速部署上车指南
date: 2017-02-16 03:14:17
tags: [https, debian, certbot, letsencrypt]
---

2017年的今天，全站HTTPS几乎已经成为一个网站的基本要求之一，最新版本的Chrome和Firefox都已经开始对HTTP站点显示“不安全”提示。由于HTTP使用明文传输，不仅会被嗅探报文，甚至会遭遇中间人攻击，不需要很多经验的攻击者就可以截取你的权限信息并肆意更改你的账户。更重要的是，随着HTTPS被大力推广，越来越多免费证书可供选择，HTTPS带来的性能负担也逐渐被削弱以及从考虑重心中排除。本文以部署Debian Jessie上的Node.js应用为例，提供快速HTTPS部署方案，我们采用的是[Let's Encrypt](https://letsencrypt.org/)提供的免费证书，一次获取可以使用3个月。

## 环境说明

我的服务器使用的发行版是Debian Jessie，服务器程序是Nginx。一般而言，不同的发行版只是安装certbot的方法不一样，其他应该是一样的。

## Nginx的初始配置

在没有启用HTTPS的情况下，相信读者对下面的Nginx配置非常熟悉，它可以为域名myapp.perqin.com设置一个监听3000端口的Node.js应用：

```
server {
	listen		80;
	server_name	myapp.perqin.com;
	
	location / {
		proxy_set_header	X-Real-IP $remote_addr;
		proxy_set_header	X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header	X-NginX-Proxy true;
		proxy_pass		http://localhost:3000/;
		proxy_cache_bypass	$http_upgrade;
		proxy_redirect		off;
	}
}

```

除此以外，Nginx还有一个用于静态站点的默认配置，我们也会用到：

```
server {
    listen       80;
    server_name  localhost;
    
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    # redirect server error pages to the static page /50x.html
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

以上就是部署HTTPS之前的服务器状态，假设Nginx和Node.js应用都已经可以正常使用，现在我们准备刷卡上车了！

## 配置静态站点

简单科普一下（如有错误请指出）：

>  HTTPS使用了TLS，它涉及了对称和非对称加密，对称的主密钥用于加密HTTP请求，非对称密钥用于加解密主密钥。网站的证书由CA签发，用户（浏览器等）通过检验证书是否的确由CA签发，就可以确定其中包含的公钥是否属于该域名，并用它加密。私钥只有域名所属的服务器拥有，所以可以保证该流量只能被真实的目的服务器获取并解读。浏览器等一般都内置了各大CA的证书，因此可以检验服务器提供的证书是否由相应的CA签发。
>
>  要获得某个域名的证书，就需要向CA证明自己是该域名的拥有者，有专门的协议可以进行验证，大致的原理是域名拥有者将域名解析到自己的服务器，然后在该服务器部署一个静态网站，使得制定的目录及其内容可通过HTTP协议获取并被程序修改。简单地想，验证程序E向CA的服务器请求给myapp.perqin.com发一个证书，CA要求myapp.perqin.com在`/.well-known/abc.txt`中写上`def`。E完成该任务并告知CA，然后CA会发送请求`http://myapp.perqin.com/.well-knwon/abc.txt`，并判断里面的内容是否是`def`即可。
>
>  **注：以上内容都是我猜的。**

因此，我们需要修改myapp.perqin.com的Nginx配置，使得：

*  对于`http://myapp.perqin.com/.well-knwon/*`的请求，一律放行到静态目录，如`/usr/share/nginx/html`
*  对于其他的HTTP请求，一律返回302跳转，跳转到对应的HTTPS网址
*  对于HTTPS请求，转发到Node.js应用

于是，我们得到一下配置：

```
# HTTP - only for obtaining cert
server {
	listen		80;
	server_name	myapp.perqin.com;

	location /.well-known {
		root	/usr/share/nginx/html;
		allow	all;
	}

	location / {
		return	301 https://$host$request_uri;
	}
}

# HTTPS - serve nodejs app
server {
	listen		443 ssl;
	server_name	myapp.perqin.com;

	#ssl_certificate		/etc/letsencrypt/live/myapp.perqin.com/fullchain.pem;
	#ssl_certificate_key	/etc/letsencrypt/live/myapp.perqin.com/privkey.pem;

	location / {
		proxy_set_header	X-Real-IP $remote_addr;
		proxy_set_header	X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header	X-NginX-Proxy true;
		proxy_pass		http://localhost:3000/;
		proxy_ssl_session_reuse	off;
		proxy_set_header	Host $http_host;
		proxy_cache_bypass	$http_upgrade;
		proxy_redirect		off;
	}
}

```

注意：

*  修改相关域名、端口号为你的域名、Node.js监听端口号
*  L21-L22需要暂时注释掉，因为此时你的证书还没有生成，不注释将无法使用

接下来重启Nginx：

```
$ sudo systemctl restart nginx.service
```

## 配置certbot

certbot是Let's Encrypt推荐的工具，他会自动完成证书验证、获取、存储和定时续签任务。

英文好的同学可以直接转乘[Certbot - All Instructions](https://certbot.eff.org/all-instructions/#debian-8-jessie-none-of-the-above)阅读配置教程。

首先，我们需要为Debian添加backports仓库源，编辑`/etc/apt/source.list.d/jessie-backports.list`：

```
deb http://ftp.debian.org/debian jessie-backports main
```

保存之后运行：

```
$ sudo apt-get update
$ sudo apt-get install certbot -t jessie-backports
```

安装certbot之后，我们可以使用它的webroot插件获取证书：

```
$ sudo certbot certonly --webroot -w /usr/share/nginx/html -d myapp.perqin.com
```

上述命令中`certonly`表示只获取证书，`--webroot`制定插件，`-w`后面是你的Nginx静态目录，`-d`后面就是域名了。注意这里`myapp.perqin.com`和`perqin.com`是不一样的，如果需要两个都上车，则需要连续两次`-d subdomain.domain.tld`。

另外，由于证书、日志都分别存储在了`/etc`和`/var/log`中，因此需要root权限。

等待片刻，就能看到提示消息，此时你的证书已经躺在他给出的位置（一般也是我前面Nginx配置文件中的位置），而且自动配置了crontab计划任务，每天两次判断是否有即将过期的证书并续签。

初次配置可能会要求输入邮箱，大可放心输入，它用于你的证书要GG的时候给你发提醒。

最后，再次编辑Nginx的配置文件，去除那两行的注释并保存，然后重启Nginx服务。

现在，在浏览器输入myapp.perqin.com，你会发现你被重定向到了https://myapp.perqin.com！

## 总结

总的来说，整个过程是非常简单的，Nginx的配置对每个网站都是大同小异的，可以直接复制粘贴修改前述模板。certbot的安装也很简单，配置参数也很自然。当然，当初第一次折腾的时候我也研究了半天才得出了前面的Nginx模板，保证既能通过HTTP请求`/.well-knwon/`，又能屏蔽其他的全部HTTP请求。看完本文，你应该也能快速发车，给你的网站系上HTTPS安全带了！