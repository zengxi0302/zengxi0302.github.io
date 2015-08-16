---
layout: post
title:  "在ubuntu14.04下安装systemtap"
date:   2015-07-16 23:54:32
categories: linux
---

###*获取对应内核版本的debuginfo安装包*

首先，查询所用系统的内核版本

{% highlight ruby %}
zengxi@zengxi-ubuntu:/$ uname -r
3.13.0-24-generic
{% endhighlight %}

本人机器是ubuntu14.04，可到一下网址获取debuginfo包

<http://ddebs.ubuntu.com/pool/main/l/linux/linux-image-3.13.0-24-generic-dbgsym_3.13.0-24.46_i386.ddeb>

然后，使用`dpkg -i linux-image-3.13.0-24-generic-dbgsym_3.13.0-24.46_i386.ddeb`安装

安装完后确认/usr/lib/debug目录下有安装的文件。

###*安装systemtap*

`sudo apt-get install systemtap`

使用命令测试一下是不是安装成功

`stap -ve 'probe begin { log("hello world") exit () }'`

具体systemtap的原理和介绍可参考这篇文档

<http://www.ibm.com/developerworks/cn/linux/l-cn-systemtap3/index.html>

