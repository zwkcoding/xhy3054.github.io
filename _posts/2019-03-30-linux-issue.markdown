---
layout: post
title: linux常用小知识--数据流重定向、管道、组合键等
date: 2019-03-30 10:07:24.000000000 +09:00
img:  sword/sword2.jpg # Add image post (optional)
tag: [linux]
---

在学习linux的内容中，存在很多重要有趣有用的小知识，但是这些小知识很繁琐，每一个都写一片博客是不可能的，篇幅太短，所以这篇博客就来将这些小知识来一个大杂烩。

## 数据流重定向
要想了解数据流重定向，首先我们得知道数据的输入与输出总共有几种。
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/linux/redirection.jpg"/>
</div>
如上，命令的输入只有一种，那就是**标准输入(STDIN)**；而输出有两种，一种是**标准输出(STDOUT)**，一种是**标准错误输出(STDERR)**。标准输出(STDOUT)的内容是一个命令正确执行后返回的内容，标准错误输出(STDERR)的内容是一个命令执行过程中出现错误返回的错误信息。

> 上述的三种输入输出可以理解为三个不同的**缓冲区**，输入缓冲区只有一个，输出缓冲区有两个。

默认情况下，在终端执行命令时，**STDOUT与STDERR都是默认绑定到了屏幕上**，因此执行一个命令，无论正确执行与否，最终输出都会出现在屏幕上。**STDIN默认绑定在了键盘上**，所以我们可以通过手动输入确定输入内容。

**数据流重定向**要做的就是重新定向这三个缓冲区绑定的字符流设备。重定向方法形式如下
1. 标准输入(stdin): 代码为0，使用`<`或`<<`;

2. 标准输出(stdout): 代码为1，使用`>`或`>>`;

3. 标准错误输出(stderr): 代码为2，使用`2>`或`2>>`;

4. `>`与`>>`的区别是：前者是覆盖输入，后者是累加输入；

```bash
xhy@xhy-SVF14316SCB:~/Music$ ls
CloudMusic
xhy@xhy-SVF14316SCB:~/Music$ ls >> ls_result
xhy@xhy-SVF14316SCB:~/Music$ cat ls_result 
CloudMusic
ls_result
```

## 命令执行的连接符号`; && ||`
- `cmd1; cmd2`: 顺序执行cmd1与cmd2

- `cmd1 && cmd2`: 
    - 若cmd1执行完毕且正确执行(`$?=0`指令回传值为0)，则开始执行cmd2；
    - 若cmd1执行完毕且错误执行(指令回传值不为0)，则cmd2不执行；

- `cmd1 || cmd2`:
    - 若cmd1执行完毕且正确执行(`$?=0`指令回传值为0)，则cmd2不执行；
    - 若cmd1执行完毕且错误执行(指令回传值不为0)，则开始执行cmd2；

## 管道命令(pipe)
首先说明一下，管道命令要实现的功能如下图，其实就是对于输入进行一系列的连续处理。
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/linux/redirection.jpg"/>
</div>
关于管道命令，可以总结如下几点：
1. 管道命令用`|`来连接命令，实现将前一个命令的stdout变成后一个命令的stdin；

2. 管道命令默认是对stdout起作用，对于stderr不起作用

3. 如果一个命令不能接受stdin，则这个命令不能作为管道中的后置命令

```bash
xhy@xhy-SVF14316SCB:~/book$ ll
total 16
drwxrwxr-x  4 xhy xhy 4096 3月  20 15:34 ./
drwxr-xr-x 41 xhy xhy 4096 3月  30 20:11 ../
drwxrwxr-x 10 xhy xhy 4096 3月  20 15:33 book_note/
drwxr-xr-x  2 xhy xhy 4096 3月  20 23:31 工具书/
xhy@xhy-SVF14316SCB:~/book$ ll | grep book_note
drwxrwxr-x 10 xhy xhy 4096 3月  20 15:33 book_note/
```

> 其实管道命令涉及到了数据流重定向的功能。如果想让stderr可以被管道命令使用，需要使用数据流重定向，将`2>`变成`1>`

## 回车与换行的区别
1. `\r`:   回车，打印效果是将输出重定位到**本行**的开头

2. `\n`:   换行，打印效果是将输出重定位到**下一行**的开头

## 常用组合键
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/linux/combine_input.png"/>
</div>

## bash环境中的特殊字符
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/linux/special_character.png"/>
</div>

## 通配符
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/linux/common_character.png"/>
</div>


# 参考资料
- [1] 《鸟哥的linux私房菜》；

