---
layout: post
title: 简单理解应用AWK
date: 2018-04-22 08:07:24.000000000 +09:00
img: how-to-start.jpg # Add image post (optional)
tags: [linux, 文本处理]
---

## what's ***AWK***
> AWK是一种优良的文本处理工具，Linux及Unix环境中现有的功能最强大的数据处理引擎之一。这种编程及数据操作语言（其名称得自于它的创始人阿尔佛雷德·艾侯、彼得·温伯格和布莱恩·柯林汉姓氏的首个字母）的最大功能取决于一个人所拥有的知识。——维基百科

由上可知，**AWK**是一种强大的用于文本分析处理的工具。很多人简单的把它看作**linux**下的一个命令，这种观点是浅显的，**AWK**实际上也是一个脚本解释器（与shell一样），通过下文，我们可以逐渐发现**AWK**的强大。

## ***AWK：***基本知识汇总
实际上AWK的确拥有自己的语言：AWK程序设计语言，三位创建者已将它正式定义为“样式扫描和处理语言”。AWK提供了极其强大的功能：可以进行正则表达式的匹配，样式装入、流控制、数学运算符、进程控制语句甚至于内置的变量和函数。它具备了一个完整的语言所应具有的几乎所有精美特性。

AWK在处理文本时，一次读入一个记录，每行为一条记录。每条记录会划分成多个域，最终将一个文件规划成一个表格类型的数据结构。

### AWK的内置变量
- $0：当前记录，这个变量存放了当前整行的内容。
- $1~$n：当前记录的第n个域的内容。（由FS进行分隔）
- FS：域分隔符”。其默认值为“空白字符”，即空格和制表符。FS可以替换为其它字符，从而改变域分隔符。
- NF：当前记录中的域个数，也就是总列数。
- NR：已经读出的记录的条数，也就是行号，如果多个文件的话，这个变量会一直累计。
- FNR：也是行号，不过刚开始读每个文件时会清零，因此只代表这条记录在当前文件中的行号。
- RS：“记录分隔符”。默认为换行符。
- OFS：“输出域分隔符”，默认空格。
- ORS：“输出记录分隔符（行）”默认换行符。
- FILENAME：当前输入文件名。


### ***AWK：***作为一个命令用
假设有一段文本`test.log`，文本内容如下：

	Proto    Recv-Q   Send-Q   Local-Address      Foreign-Address        State
	tcp      0        0        0.0.0.0:3306       0.0.0.0:*              LISTEN
	tcp      0        0        0.0.0.0:80         0.0.0.0:*              LISTEN
	tcp      0        0        127.0.0.1:9000     0.0.0.0:*              LISTEN
	tcp      0        0        coolshell.cn:80    124.205.5.146:18245    TIME_WAIT
	tcp      0        0        coolshell.cn:80    61.140.101.185:37538   FIN_WAIT2
	tcp      0        0        coolshell.cn:80    110.194.134.189:1032   ESTABLISHED
	tcp      0        0        coolshell.cn:80    123.169.124.111:49809  ESTABLISHED
	tcp      0        0        coolshell.cn:80    116.234.127.77:11502   FIN_WAIT2
	tcp      0        0        coolshell.cn:80    123.169.124.111:49829  ESTABLISHED
	tcp      0        0        coolshell.cn:80    183.60.215.36:36970    TIME_WAIT
	tcp      0        4166     coolshell.cn:80    61.148.242.38:30901    ESTABLISHED
	tcp      0        1        coolshell.cn:80    124.152.181.209:26825  FIN_WAIT1
	tcp      0        0        coolshell.cn:80    110.194.134.189:4796   ESTABLISHED
	tcp      0        0        coolshell.cn:80    183.60.212.163:51082   TIME_WAIT
	tcp      0        1        coolshell.cn:80    208.115.113.92:50601   LAST_ACK
	tcp      0        0        coolshell.cn:80    123.169.124.111:49840  ESTABLISHED
	tcp      0        0        coolshell.cn:80    117.136.20.85:50025    FIN_WAIT2
	tcp      0        0        :::22              :::*                   LISTEN

输入命令`awk '$3>0 || NR==1 {print $0}' test.log > record.log`,查看`record.log`中输出如下：

	Proto Recv-Q Send-Q Local-Address          Foreign-Address             State
	tcp        0   4166 coolshell.cn:80        61.148.242.38:30901         ESTABLISHED
	tcp        0      1 coolshell.cn:80        124.152.181.209:26825       FIN_WAIT1
	tcp        0      1 coolshell.cn:80        208.115.113.92:50601        LAST_ACK

输入命令`awk '$6 ~ /TIME_WAIT|State/ {printf"%02d %10s %30s %20s \n",NR,$1,$4,$6}' test.log > record.log`，查看`record.log`中输出如下：

	01      Proto                  Local-Address                State 
	05        tcp                coolshell.cn:80            TIME_WAIT 
	11        tcp                coolshell.cn:80            TIME_WAIT 
	15        tcp                coolshell.cn:80            TIME_WAIT 
	
如上我们可以发现，是如下模式：

        pattern { action }

上面是AWK的基础命令模式，AWK将对输入的文本进行逐记录（行）处理，在遇到符合`pattern`的记录时，执行`action`。所以，如果有10行符合pattern，则action会执行10次。



### ***AWK：***脚本编写

有`score.log`，内容如下：

	Marry   2143 78 84 77
	Jack    2321 66 78 45
	Tom     2122 48 77 71
	Mike    2537 87 97 95
	Bob     2415 40 57 62

编写一个后缀名为`.awk`的脚本，内容如下：

	#!/bin/awk -f
	#运行前
	BEGIN {
	    math = 0
	    english = 0
	    computer = 0
	 
	    printf "NAME    NO.   MATH  ENGLISH  COMPUTER   TOTAL\n"
	    printf "---------------------------------------------\n"
	}
	#运行中
	{
	    math+=$3
	    english+=$4
	    computer+=$5
	    printf "%-6s %-6s %4d %8d %8d %8d\n", $1, $2, $3,$4,$5, $3+$4+$5
	}
	#运行后
	END {
	    printf "---------------------------------------------\n"
	    printf "  TOTAL:%10d %8d %8d \n", math, english, computer
	    printf "AVERAGE:%10.2f %8.2f %8.2f\n", math/NR, english/NR, computer/NR
	}

在终端输入`awk -f score.awk score.txt`，输出结果如下：

	NAME    NO.   MATH  ENGLISH  COMPUTER   TOTAL
	---------------------------------------------
	Marry  2143     78       84       77      239
	Jack   2321     66       78       45      189
	Tom    2122     48       77       71      196
	Mike   2537     87       97       95      279
	Bob    2415     40       57       62      159
	---------------------------------------------
	  TOTAL:       319      393      350 
	AVERAGE:     63.80    78.60    70.00

如上便是AWK的脚本方式，如果想要将脚本嵌入shell中，只需要把AWK代码放入`''`中便可以了，而且这样做不需要`-f`参数。方式：`awk '代码'`
