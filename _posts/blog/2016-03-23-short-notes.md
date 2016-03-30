---
layout: post
title: 简要笔记
description: MySQL zookeeper redis nginx host terminus-license ssh github
category: blog
---

## 简要启动命令

###1、启动MySQL
打开iTerm，输入如下命令：

    $ MySQL.server start

###2、启动zookeeper
接着可以输入zookpeer启动命令：

    $ sudo zkServer start

###3、启动redis
根据电脑redis配置文件，输入一下命令：

	$ redis-server /usr/local/etc/8888.conf

###4、启动nginx

	$ sudo nginx

###5、配置host

	$ sudo vi /etc/hosts
###6、配置terminus-license

	$ cd terminus/terminus/license
	$ cd license-gen/target
	$ vi ../../README.md
	$ java -jar keygen stronglong.me 2016-1-1

###7、开分支提交

	$ git flow init
	$ git flow feature start <branch-name>

###8、配置软链接

	$ cd terminus
	$ terminus: cd assets
	$ assets: ls
	$ ........
	$ assets: ln -s ../yanma<前端名>/public yanma

###9、elasticsearch

    $ /usr/local/var/elasticsearch
    $ elasticsearch

## 配置和使用Github

###1、检查SSH keys的设置
首先检查电脑现有的ssh key:

	$ cd ~/.ssh

如果显示"No such fie or directory"，跳到第三部，否则继续。

###2、备份和移除原来的ssh key设置
因为已经存在key文件，所以需要备份旧的数据并删除：

	$ ls
	config id_rsa id_rsa.pub known_hosts
	$ mkdir key_backup
	$ cp id_rsa* key_backup
	$ rm id_rsa*

###3、生成新的SSH key
输入一下的代码，就可以生成新的key文件，后面只需要默认设置就好，所以当需要输入文件名的时候，回车就好。

	$ ssh-keygen -t rsa -C "strong@long.com"
	Generating public/private rsa key pair.
    Enter file in which to save the key (/Users/your_user_directory/.ssh/id_rsa):<回车就好>

然后系统会要求输入加密串：

	Enter passphrase (empty for no passphrase):<输入加密串>
	Enter same passphrase again:<再次输入加密串>

最后看到这样的界面，就成功设置ssh key了：
![ssh key success](/images/githubpages/ssh-key-set.png)

###4、添加SSH key 到Github：
在本机设置SSH Key之后，需要添加到Github上，以完成SSH链接的设置。

用文本编辑工具打开id_rsa.pub文件，如果看不到这个文件，需要设置显示隐藏文件。准确的饿复制这个文件的内容，才能保证设置成功。

在Github的主页上点击设置按钮：
![github account setting](/images/githubpages/github-account-setting.png)

选择SSH Keys项，把复制的内容粘贴进去，然后点击Add Key按钮即可：
![set ssh keys](/images/githubpages/bootcamp_1_ssh.jpg)

PS：如果要设置多个Github帐号，可以参看这个[多个github帐号的SSH Key切换](http://omiga.org/blog/archives/2269)，如果通过了文章的所述配置了host，那么多个帐号下面的提交会是一个人的，所以需要通过命令`git config --global --unset user.eamil`删除用户账户设置，在每一个repo下面使用`git config --local user.email'你的github邮箱@mail.com'` 命令单独设置用户账户信息

###5、测试一下
 可以输入下面的命令，看看设置是否成功，`git@github.com`的部分不做修改：

	$ ssh -T git@github.com

如果是下面的反应：

	The authenticity of host 'github.com (207.97.227.239)' can't be established.
	RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
    Are you sure you want to continue connecting (yes/no)?

没事，继续输入`yes`就好，然后会看到：

	Hi <em>username</em>! You've successfully authenticated,but GitHub does not provide shell access.

###6、设置个人帐号信息
现在已经可以通过SSH链接到Github了，还有一些个人信息需要完善。

Git会根据用户的名字和邮箱来记录提交。Github也是用这些信息来做权限的处理，输入下面代码进行个人信息的设置把名称和邮箱替换为自己的。

	$ git config --global user.name "你的名字"
	$ git config --global user.email "your_email@youremail.com"

##结语
简短记录工作中所需要的知识，有的是自己的，有的则是借鉴前辈的优秀总结。



[StrongL]:    http://stronglong.me  "StrongL"
