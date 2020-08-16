---
layout: post
title: pycharm远程调试
date: 2020-08-14 10:07:24.000000000 +09:00
img:  one-piece/one-piece26.jpg # Add image post (optional)
tag: [tools]
---

我们在本地windows使用各种IDE可以很方便的调试代码，但是有的项目对内存或者显存需求量很大，本地的机器很难满足。实验室刚好有一个linux服务器，内存够大，gpu够给力，不过我没有在服务器上安装IDE的权限，于是就鼓捣了一下pycharm的远程调试。

# pycharm远程调试

pycharm的远程调试其实就是利用ssh先将本地工程拷贝到远程服务器上，然后在远程服务器上运行，所以我们需要给pycharm提供以下的信息：

1. 远程服务器ip，用户名，密码；

2. 工程使用的python解释器（一般是远程服务器上anaconda的一个环境）；

3. 本地工程文件夹与远程服务器上的工程文件夹的对应关系；

## 配置步骤

- 首先使用pycharm打开本地的工程目录；

- 依次打开`File`->`Setting`->`Project:工程名`->`Project Interpreter`；

<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/pycharm/pycharm1.PNG"    width="800" height="280"/>
</div>

- 选择`Project Interpreter`最右侧的那个设置按钮点`add`，新界面选择`SSH Interpreter`;

<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/pycharm/pycharm2.PNG"    width="800" height="310"/>
</div>

- 在`New server configuration`中输入ip和用户名，点击`Next`，输入密码连接后;

<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/pycharm/pycharm3.PNG"    width="800" height="550"/>
</div>

- 其中`Interpreter`是远程服务器中你想使用的python环境地址，`Sync folders`是本地和远程的工程文件夹对应关系，两个同名文件夹，设置完后选择`Finish`;

- 然后pycharm会自动将本地工程复制到远程服务器上。

- 复制完毕后，要想远程调试还需要`Run`->`edit Configurations`，然后在这个界面中选择运行和调试时使用的python解释器`Python interpreter`，还有通过`Parameters`设置运行时的参数输入。

<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/pycharm/pycharm4.PNG"    width="800" height="400"/>
</div>

- 大功告成，可以调试了。

- 如果想对远程服务器进行管理，可以`Tools`->`Deployment`->`configure`进行改名、新建、删除等，其中`Encoding for client-server communication`是与服务器通信的编码方式，可以设置成`UTF-8`(如果包含中文的话)；

<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/pycharm/pycharm5.PNG"    width="800" height="800"/>
</div>