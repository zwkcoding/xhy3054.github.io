---
layout: post
title: Git与Github简要教程
date: 2018-04-12 13:07:24.000000000 +09:00
img: github-for-atom.png # Add image post (optional)
tags: [tools]
---



## what's Git and what's Github
Git是一种分布式版本控制系统，具体学习可以参考[廖雪峰的git教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000),讲解清晰易懂，本文主要介绍如何运用git与github进行基本上手操作。

GitHub是通过Git进行版本控制的软件源代码托管服务。



## 安装git
```bash
$ sudo apt-get install git
```



## git与github基本使用


### git与github远程库交互
本地创建版本库**learngit**
```bash
$ mkdir learngit
$ cd learngit
$ git init
```

编辑完文件**readme**添加到仓库并提交
```bash
$ git add readme
$ git commit -m 'first commit'
```

如果是第一次使用git，那么`git commit`可能报错，所以需要你配置一些个人信息
```bash
$ git config --global user.email "you@example.com"
$ git config --global user.name "Your Name"
```

上传到github,首先登陆自己的github帐号，然后新建一个**repository**并取名为**learngit**,然后执行下面命令
```bash
$ git remote add origin https://github.com/xhy3054/learngit.git  #与远程库建立链接,只有第一次上传需要执行
$ git push -u origin master
```

克隆远程库到本地
```bash
$ git clone https://github.com/xhy3054/learngit.git  #与远程库建立链接
```

远程库修改过，更新本地库（将远程库拉下来与本地库合并）

	git pull #此命令用时需保证本地库没有修改未提交，

查看本地分支列表

	git branch

查看本地仓库工作状态

	git status

查看本地库关联的远程库（即push与pull推送与拉取的那个远程库）

	git remote -v

修改本地库关联的远程库

	git remote -v set-url origin https://github.com/xhy3054/ssdcaffe.git

### git版本管理

查看历史版本

	$git log
	commit 85c18305998dc282973a21513aa1893567a52e33
	Author: xhy3054 <1076170656@qq.com>
	Date:   Mon May 7 14:31:42 2018 +0800
	
	    buma
	
	commit fd2bc5b0c3d2bbb06a51ec757c34f27a20ca1e8f
	Author: xhy3054 <1076170656@qq.com>
	Date:   Sat May 5 20:51:53 2018 +0800
	
	    add a image

回退到版本`add a image`

	git reset --hard fd2bc5

查看当前分支

	git branch

新建分支`dev`

	git branch dev

切换到分支`dev`

	git checkout dev

如果`dev`上的工作想合并到`master`

	git checkout master
	git merge dev

删除分支`dev`

	git branch -d dev


其他git命令请[查询](http://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html).
