---
layout: post
title: APT包管理简析
date: 2018-05-01 10:07:24.000000000 +09:00
img:   your_name.jpg # Add image post (optional)
tags: [linux]
---
## Advanced Packaging Tool(APT)
>高级打包工具（英语：Advanced Packaging Tools，缩写为APT）是Debian及其派生发行版的软件包管理器。APT可以自动下载，配置，安装二进制或者源代码格式的软件包，因此简化了Unix系统上管理软件的过程。——维基百科

Debian是ubuntu的母版系统，Advanced Packaging Tools(APT)是各种基于Debian的发行版通用的一种**软件包管理工具**。借助这个包管理工具，用户可以方便的为自己电脑上的Linux安装各种程序软件包。注意：此处的APT与shell的apt命令是不一样的哦

## ubuntu软件源的选择
Ubuntu是用的APT进行包管理的,而这种管理方式是基于软件源的。软件源，顾名思义，就是各种软件包的集合。用户在软件源里查找自己需要的包进行安装十分方便。全世界有很多的Ubuntu软件源服务器，对于普通用户来讲，每个软件源上的包资源基本都是一样的，因为自己想找的包无论在哪个软件源上，基本都会存在。安装完Ubuntu后，其默认的软件源是Ubuntu官方提供的软件源服务器。由于这个服务器是在美国，而我国又有一些网络因素存在，使得用这个软件源安装软件时速度十分的慢。想要解决这个问题，换一个软件源就好了。

如何更换软件源呢？系统的软件源列表就是`/etc/apt/sources.list`文件，打开我们会发现里面大多是如下形式：

	deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted
	deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial multiverse universe main restricted 

> `mirrors.tuna.tsinghua.edu.cn`代表我使用的是清华的软件源，修改软件源其实只要修改这个域名就好了。上面每一项的含义是什么呢？具体apt又是怎么工作的呢？

首先，每个软件源服务器上都会将各个版本各种类型的ubuntu的软件库列表区分开来，在这里指明了软件源列表的网络位置，其中一些单词的含义下面列出。（其中，`deb-src`代表的是二进制包。）

1. xenial - 代表适用ubuntu16.04系统的软件。（一般此项xenial代表的不同版本系统存储路径下会有以下Main、Universe等路径）
2. Main - Canonical支持的免费和开源软件。
3. Universe - 社区维护的免费和开源软件。
4. restricted - 设备的专有驱动程序。
5. Multiverse - 受版权或法律问题限制的软件

上述这些进入Ubuntu桌面的***设置-软件和更新***可以看的更加直观方便些。用户可以根据自己的需要添加自己可能会用到的软件服务选项。
![gui-apt]({{site.baseurl}}/assets/img/apt/apt-gui.PNG)


上面其实是指明了清华的源上可用软件列表的网络存储位置，软件列表存储的信息有哪些软件包可用、以及去哪里可以获取它。如果想要安装一个软件，就可以到这个网络位置上对软件列表进行索引，如果能索引到这个软件包便能定位它并下载安装。但是每次安装软件都要去访问这个网络位置是很耗时费力的，因此系统实际的做法是会在本地存储一份源的软件列表，在要安装软件时直接索引本地的软件列表。所以，在你更换了自己的软件源后一定要记得`sudo apt update`，这个命令会重新从`source.list`上的网络位置上更新本地的软件列表，不更新的话你的新软件源是不生效的。同时为了始终安装最新的软件包，建议每隔一段时间便执行一次`sudo apt update`，这样可以使得自己本地的软件列表的软件版本一直是最新的。

想要替换软件源，可以直接修改`sources.list`，也可以在`系统设置/软件和更新`里进行选择。替换哪个软件源呢？其实也很容易选择，`ping`一下看看速度，选择最快的一个便好，下面是我选择的过程。因为校园网的问题，不知为什么ping不通阿里的源，然后可以ping通中科大与清华的源，因为清华的源速度更快，我选择了它。友情提醒：用虚拟机的朋友，直接在虚拟机下ping，由于虚拟机的网络是在本机的局域网内的重映射，没有进行特殊设置的话，一般只能ping通主机，是ping不通任何软件源的。所以我ping是在本机Windows上进行的。

![ping_ustc]({{site.baseurl}}/assets/img/apt/ping_ustc.png)

清华比中科大快一些

![ping_tsinghua]({{site.baseurl}}/assets/img/apt/ping_tsinghua.png)

> 注意：`apt update`与`apt upgrade`是不一样的，前一个只是**更新本地存储的软件列表**，后一个会将本地系统安装的软件与软件列表中的版本进行比对，如果有更新的版本，就会**升级软件**。
## apt命令与apt-get命令
上文提到了Debian及其派生发行版使用的是Advanced Packaging Tool(APT)进行包管理的。同时，Debian及其派生发行版也提供了一些不同的工具用来与APT进行交互，使得用户可以进行包的安装与卸载。而`apt-get`是系统提供用户使用来与APT进行交互，使得用户可以实现安装、删除、更新软件的命令行工具。相似的命令还有`apt-cache`与`apt-config`，另外还有Aptitude（既有图形界面又有命令行选项）。这三个命令里都会有一些用户特别常用的包管理命令，说的直白点，就是用户最常用的命令分散在了**apt-get**、**apt-cache**、**apt-config**里。它们三个每个里面大部分的命令用户都不会用到。

到Ubuntu16.04里，出现了`apt`命令。**apt**里包含了原本`apt-get`、`apt-cache`与`apt-config`里常用的命令和一些自己的命令，更加方便用户的使用。



