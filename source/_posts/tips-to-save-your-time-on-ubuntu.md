---
title: 加快墙内网速提升Ubuntu使用效率的TIPS
date: 2016-08-09 01:03:50
tags: [Shadowsocks, Ubuntu, npm, HA-Proxy]
---
众所周知（？），在国内想要访问某些网站是很慢的，然而也许是误伤的缘故，很多不在此列表的网站访问速度也很慢，shadowsocks的配置方法我已经在[重装Ubuntu系列：Shadowsocks配置](/2016/04/18/ubuntu-reinstallation-series-shadowsocks-configurations/)里面提到过，这里再对Ubuntu做一些简单的配置，充分利用已有的资源。

# 使用npm镜像
之前从学校回家之后，网络莫名抽风，`npm install`死活卡着不动，万般无奈之下去谷歌了一下npm的镜像，发现过内有两个镜像，淘宝提供的和cnpm提供的。后者似乎进行了负载均衡，国内用户会通过淘宝的镜像下载（不是很清楚，详见[官网](https://cnpmjs.org/)），因此选择了它。
配置方法还是很简单的，直接在Home目录下新建一个`.npmrc`的文件，并写入如下内容：
```
registry=http://r.cnpmjs.org/
```
然后重新打开命令行即可。
值得一提的是，cnpm还提供了一个`cnpm`包，用于自动从cnpm镜像下载，不过好像和npm的表现不太一样，所以最终没有用它。

# 使用教育网apt-get镜像
这个就不多解释了，直接发车。打开Dash面板搜索`Software and Updates`，然后打开`Ubuntu Software`标签页下的`Download from:`列表，选择`'Other...`，从弹出的对话框中列表里的China里选择一个即可。

# 使用国内云服务器和HA-Proxy做Shadowsocks中继
校园网的环境真是尿崩啊……ping我US的Shadowsocks服务器居然丢包达到了25%，谷歌一番之后，发现了神器HA-Proxy，它可以作为TCP和HTTP的中继，于是我尝试在我的国内云服务器ping我的Shadowsocks服务器，发现延迟在200ms左右，不丢包。从我本地ping国内云服务器，延迟10ms以内，不丢包。可以，很强势！于是，方案浮出水面：在国内云服务器上安装HA-Proxy，将A端口转发到Shadowsocks服务器的B端口，然后将本地的Shadowsocks服务器IP改为国内云服务器IP，端口改为A。

 * 在国内云服务器安装HA-Proxy

   ```
   sudo apt-get install haproxy
   ```

 * 修改国内云服务器上的`/etc/haproxy/haproxy.cfg`，将`defaults`内的`mode`改为`tcp`，并在后面添加前后端配置
   
   ```
   frontend ss-in
   	bind *:A
   	default_backend ss-out
   
   backend ss-out
   	server server1 1.2.3.4:B maxconn 20480
   ```
   
   其中1.2.3.4就是你的Shadowsocks服务器IP。

 * 保存编辑，重启服务

   ```
   sudo service haproxy restart
   ```

客户端的配置略去，需要注意如果有多个端口的话，上述配置文件可以省略端口B。另外，HA-Proxy不支持UDP中继，这可能会造成一些影响，因为Shadowsocks默认开启UDP代理，这可能导致UDP包被丢弃，等我遇到坑了再来田- -