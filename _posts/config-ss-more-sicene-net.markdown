---
layout: post
title:  "配置 Shadowsocks 让多客户端科学上网~"
date:   2016-03-04 09:00:00
categories: 技术分享
tags: [科学上网, 技术分享]
---

最近一直再找一些可以用手机翻墙的工具或者免费的vpn账号，但是效果都很差，其实去年就购买了Shadowsocks的服务作为查找资料的工具，不过苦于手机不能上外网，所以今天共享一下这个教程，顺便记录一下！<!--more-->

windows 下用 shadowsocks 做局域网很简单，只要勾选就行了。下面介绍 mac 上的配置。

shadowsocks 的 mac 安装，自己下包安装就好了。

shaodowsocks 的 mac 配置需要 privoxy
#### 1.安装

```
$ brew install privoxy  
```

![QQ20160304-1.png](http://7xqhcq.com1.z0.glb.clouddn.com/config-ssss-545755-b92071ea13e3bad5-1.png)
安装完成后会提示加入到开机启动里

```
To have launchd start privoxy at login:  
  ln -sfv /usr/local/opt/privoxy/*.plist ~/Library/LaunchAgents  
Then to load privoxy now:  
  launchctl load ~/Library/LaunchAgents/homebrew.mxcl.privoxy.plist  
Or, if you don't want/need launchctl, you can just run:  
  privoxy /usr/local/etc/privoxy/config  
```

然后按照上面终端的提示执行，将privoxy加入随机启动

![QQ20160304-2.png](http://7xqhcq.com1.z0.glb.clouddn.com/config-ssss-545755-caa708b7abe4c66f-2.png)

#### 2.配置

```
$ vim /usr/local/etc/privoxy/config  
#修改下面的  
listen-address  0.0.0.0:8118    #8118是 privoxy 的监听端口  
forward-socks5   /   127.0.0.1:1080.    #1080是 shadowsocks 的监听端口  
```

或者用这种方法直接写入

```
$ echo 'forward-socks5 / localhost:1080 .' >> /usr/local/etc/privoxy/config
$ echo 'listen-address 0.0.0.0:8118' >> /usr/local/etc/privoxy/config
```

重启 privoxy，看到8118监听端口，即成功

```
$ netstat -an | grep 8118  
tcp4       0      0  *.8118                 *.*                    LISTEN  
```

![QQ20160304-3.png](http://7xqhcq.com1.z0.glb.clouddn.com/config-ssss-545755-4d46fc9058da6423-3.png)

以上电脑端的配置就完成了~

测试一下，手机 wifi http 代理下，选择手动，然后填写 mac 的 IP，192.168.2.177:8118，访问下谷歌，OK！

![QQ20160304-3.png](http://7xqhcq.com1.z0.glb.clouddn.com/config-ssss-545755-4f0715608d1bab67-4.png)

> 有多少人和我一样，坐在不足10平米的空间里，看着书里九万五千公里的绚丽。又或是和我一样，拥有一颗比九万五千公里还辽阔的心，却坐在不足一平米的椅子上。

本站文章如无其他特殊说明，均为原创，转载请注明出处。

#### 参考链接：
[mac 配置 Shadowsocks 做局域网共享](http://liuhonghe.me/mac-set-shadowsocks-share-lan.html)

