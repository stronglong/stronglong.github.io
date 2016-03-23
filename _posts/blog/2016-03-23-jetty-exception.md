---
layout: post
title: 简要笔记
description: MySQL zookeeper redis nginx host terminus-license ssh github
category: blog
---

## BindException
今天IDEA突然挂了，重启之后在启动本地项目admin，出现了如下问题：
<p><code> Jetty server - java.net.BindException: 127.0.0.1：9001  Address already in use </code></p>

以至项目跑不起了.........

浏览SOF之后，原来是端口占用了。

于是乎就有了接下来的操作：

	$ lsof -i :9001
	COMMAND   PID     USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
	java    75284 terminus   64u  IPv6 0xb794fee8b3c7c127      0t0  TCP *:etlservicemgr (LISTEN)

得到了被占用端口的PID：75284

查看一下jetty进程，看看里面是否有75284：

	$ ps aux | grep jett

执行命令得如下结果：
![jetty thread](/images/other/jetty-thread.png)

进程里果然存在75284，so

	$ kill -9 75284

最后再次查看jetty进程，发现75284已消失

重启项目，ok








[StrongL]:    http://stronglong.com  "StrongL"
