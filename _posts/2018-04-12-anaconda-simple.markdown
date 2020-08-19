---
layout: post
title: Anaconda简要教程与jupyter book的安装
date: 2018-04-12 11:07:24.000000000 +09:00
img: anaconda.jpg # Add image post (optional)
tags: [tools]
---


### what's Anaconda

**Anaconda**,是用于python编程环境管理的一个工具。它让你可以为每一个项目单独的构建自己的python编程环境，因此很好的解决了不同项目之间的串扰问题。并且使得你的工作更加清晰有条理。


### install Anaconda
从[官网](https://www.anaconda.com/download/)下载安装文件，在linux下是一个类似`Anaconda2-5.1.0-Linux-x86_64.sh`文件，安装只需运行
```bash
$ bash Anaconda2-5.1.0-Linux-x86_64.sh 
```


### Anaconda启用与停用
安装的过程中会有两次让你确认的交互，第一次是让你同意他的协议，输入yes即可；第二次是问你是否将anaconda/bin添加到环境变量中，一般来说点击yes是没问题的，这样安装完，就直接和你的python3命令绑定了。如果这一步输入了no，相当于默认没有启动anaconda,需要去环境变量中添加，方法如下：

1. 打开终端并输入`gedit ~/.bashrc`

2. 在`.bashrc`文件末尾输入`export PATH=/home/xhy/anaconda3/bin:$PATH `,路径换为**自己的**，然后保存。

3. 终端输入`source~/.bashrc`，使得立即生效。

若想停用anaconda，方法如下：

1. 打开终端并输入`gedit ~/.bashrc`

2. 找到刚才添加的那行路径注释掉

3. 终端输入`source~/.bashrc`，使得立即生效。

4. 重开一终端即可看到效果。


### 使用Anaconda

列出已有的环境
```bash
$ conda env list
```

基于**python2**创建一个名为**tensorflow**的环境
```bash
$ conda create -n tensorflow python=2 
```

进入环境**tensorflow**（source可用conda替换）
```bash
$ source activate tensorflow
```

列出此**python**环境下已有的包
```bash
$ conda list
#或者 
$ pip list
```

查询某一个包(比如numpy)的版本信息
```bash
$ conda search numpy
```


此**python**环境下安装**tensorflow**（其他包类似）（也可以用**pip**安装）
```bash
$ conda install tensorflow 
#也可以pip管理
$ pip install tensorflow
```

卸载包
```bash
$ conda remove tensorflow
#如果pip管理
$ pip uninstall tensorflow
```

离开**tensorflow**环境（source可用conda替换）
```bash
$ source deactivate
```

删除**tensorflow**环境
```bash
$ conda env remove -n tensorflow
```

如果换了一台新电脑，想要将旧电脑上配置的这个**tensorflow**环境搬过去，先将此环境下的配置信息导出到一个`.yaml`文件中
```bash
$ conda env export > ~/environment.yaml 
```

然后在新电脑上新建**tensorflow**环境，然后根据`.yaml`文件更新环境
```bash
$ conda env update -f /path/to/environment.yml 
```


### anaconda的小毛病
- 如果你用pip安装的包发现也装在了系统Python下，是因为使用了系统python的pip，指定路径就好了`/home/xhy/anaconda3/bin/pip install 包名 `

- anaconda并不是完美的，虽然anaconda提供conda与pip方式在独立的env下安装Python包，但是我们知道有些包一般是用apt安装的。比如说pillow包是一个常用的Python图像处理包。但是一般安装这个包是用`sudo apt-get install python-imaging`命令安装。而apt安装的Python包默认安装在系统Python目录下，所以在anaconda的此env下的	Python并不能import到（当然你可以在import时指定路径我猜测），于是就很烦，查了一下原来这种情况也是可以安装的，虽然直接Pip和conda都找不到python-imaging包，但是可以用命令`conda install -c anaconda pillow `安装。（可以google下anaconda如何安装××包，前提你得知道包名）


## 安装jupyter lab

- 安装：

```bash
conda install -c conda-forge jupyterlab
# 或者
pip install jupyterlab
```

- 启动服务

	jupyter lab --ip=0.0.0.0 --port=8080

> windows的话直接安装anaconda，然后在navigator里便可以找到jupyter