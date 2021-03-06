---
layout: post
title: 互联网分层结构实现
date: 2018-06-21 10:07:24.000000000 +09:00
img:   network.jpg # Add image post (optional)
tags: [计算机网络]
---

本科时不是计算机专业，虽然学过计算机网络之类的课程，但是不上心，也没学懂。

自己看了些计算机网络方面的资料也做过些项目，算是一知半解，前些日子读了[阮老师的一篇博客](http://www.ruanyifeng.com/blog/2012/05/internet_protocol_suite_part_i.html)，真是通俗易懂。趁着现在有时间，边回忆边写出自己的学习笔记。

## 一、五层模型
互联网的分层有 OSI 的七层模型和 TCP/IP 的五层模型，本质上实现的功能大同小异，只不过五层模型更加精简一些。这篇博客就是在五层模型的基础上进行分析。 

![tcp_model]({{site.baseurl}}/assets/img/network/tcp_model.png)

如上图所示，其中每一层都是为了实现不同的功能，为了实现这些功能，就需要大家（通信双方）都遵守共同的规则，这些规则就是协议。互联网每一层都有很多协议，它们加起来就叫做**互联网协议（Intelnet Protocol Suite）**。五层模型中，越下面的层，越靠近硬件;越上面的层，越靠近用户。

## 二、物理层（Physical Layer）
物理层是最底下的一层，负责实现的是将电脑之间连接起来的物理实现。比如可以用光缆、电缆、双绞线、无线电波等方式。**它主要规定了网络的一些电气特性，作用是负责传送0和1的电信号**

![phy_layer]({{site.baseurl}}/assets/img/network/phy_layer.png)

## 三、链接层（Link Layer）
### 3.1 定义
首先我们知道，下一层物理层负责的是传递0和1的电信号，但是单纯的0和1是没有任何意义的，必须为它们规定解读方式：比如多少个电信号算一组？每个信号位代表了什么意义？

上述就是**链接层**的功能，它在**物理层**上方，确定0和1的分组方式，也就是说电脑在物理层收到电信号后，是在链接层将这些0和1信号转换成一个个数据包的格式。当然，它还有其他深层作用，马上会讲。

### 3.2 以太网协议
本来在早期的时候，几乎每家公司都有自己的电信号分组方式。后来逐渐的，一种叫做**以太网（Ethernet）**的协议，占据了主导地位。

在以太网协议中规定，一组电信号构成一个数据包，叫做**帧（Frame）**。每一帧分成两个部分：帧头（Head）与数据（Data）。如下：

![link]({{site.baseurl}}/assets/img/network/link.png)

其中，Head 中包含了数据包的一些说明项，比如发送者、接受者、数据类型等等; Data 则是数据包的具体内容。

Head 的长度，固定为18字节。Data 的长度，最短为46字节，最长为1500字节。因此，整个"帧"最短为64字节，最长为1518字节。如果数据很长，就必须分割成多个帧进行发送。

### 3.3 MAC地址
上面提到，以太网数据包中的 Head ，包含了发送者和接收者的信息。那么，发送者与接受者是如何标识的呢？

以太网规定，连入网络的所有设备，都必须具有“网卡接口”。数据包必须是从一块网卡，传送到另一块网卡的。网卡具有地址，就是 MAC 地址，可作为接收与发送地址。

![wangka]({{site.baseurl}}/assets/img/network/wangka.jpg)

上图为一网卡照片，每块网卡在出厂时，都会拥有一个MAC地址，长度是48个二进制位，通常用12个十六进制数表示。如下

![mac]({{site.baseurl}}/assets/img/network/mac.png)

前6个十六进制是厂商编号，后6个是该厂商的网卡流水号。有了MAC地址，就可以定位网卡与数据包的路径了。按理说，每一个 MAC 地址应该是独一无二的。但是事实并不是这样。不过一般情况下，我们很少会遇到两个网卡MAC地址一样的情况（概率很小），如果遇到了，只要两个设备不在一个子网也不会有什么影响。如果遇到了，还在同一个子网中，那么有两种解决方案：
1. 网卡厂商提供有配置程序，可以直接修改硬件 MAC 地址。
2. 操作系统会提供伪造MAC地址的方式来解决冲突。

### 3.4 因特网基于MAC地址的信息传播方式
在这里需要提前说明一下，链路层的以太网基于 MAC 地址进行通信是只能实现子网内的通信的。即两台电脑需要在一个子网内。

定义了地址只是第一步，下面一块网卡如何知道另一块网卡的MAC地址呢？

回答是通过 ARP 协议来解决这个问题的，这个协议建立在网络层协议的基础上，过会儿会在网络层部分进行解释。

其次我们需要考虑的是就算有了 MAC 地址，系统如何日才能把数据包准确送到接收方呢？

回答是以太网采用了很**原始**的一种手段，它并不是把数据包准确送到接收方，而是向本网络内所有计算机发送，让每台计算机自己进行判断，自己是否为接收方。

![boardcast]({{site.baseurl}}/assets/img/network/boardcast.png)

上图中，1号计算机向2号计算机发送一个数据包，同一个子网络的3号、4号、5号计算机都会收到这个包。它们读取这个包的"标头"，找到接收方的MAC地址，然后与自身的MAC地址相比较，如果两者相同，就接受这个包，做进一步处理，否则就丢弃这个包。这种发送方式就叫做"广播"（broadcasting）。

### 3.5 小节
讲到这里，我们发现，链路层可以将物理层传来的0和1信号转成数据包的格式，并能通过网卡的 MAC 地址，实现一个子网内多台计算机之间的数据传送了。

## 四、网络层（Network Layer）
### 4.1 网络层的由来
从上文我们知道，到链路层已经可以实现子网内设备间的通信了，但是不同子网里的设备还不能通信。因为首先我们知道一个子网不可能将所有的设备都包含进去，互联网是一个由无数子网络共同组成的一个巨型网络。

![int]({{site.baseurl}}/assets/img/network/int.png)

如图，因此首先需要找到一种方法可以区分两个机器是否属于同一个子网，如果是，那就直接**广播**方式（上一节中提到的发送方式）发送就可以了。如果不是，就采用**路由**方式发送（路由方法本博客暂不涉及）。遗憾的是，MAC 地址本身无法做到这一点，因为它只与厂商有关，与所处网络无关。

**由此导致了网络层的诞生，它的做法是引入一套新的地址，使得我们能够区分不同的计算机是否属于同一个子网络。这套地址叫做网络地址，简称网址**。

于是，网络层的诞生使得每台计算机有了两个地址，一个是 MAC 地址，一个是网络地址。它们之间没有任何联系，MAC 地址绑定在网卡上，网络地址则是管理员分配的，它们只是随机的组合在了一起。

网络地址帮助我们确定计算机所在的子网络，MAC地址则将数据包送到该子网络中的目标网卡。因此，从逻辑上可以推断，必定是先处理网络地址，然后再处理MAC地址。

### 4.2 IP协议
规定网络地址的协议叫做IP协议。它所定义的地址，被称为IP地址。

目前在我国，广泛采用的是IP协议第四版，简称 IPv4 。这个版本规定，网络地址由32个二进制位组成。
![ip]({{site.baseurl}}/assets/img/network/ip.png)

习惯上，我们会将其分成四段的十进制数表示。从0.0.0.0一直到255.255.255.255。

互联网上每一台计算机都会分配到一个 IP 地址。这个地址分成两个部分，前一部分代表网络，后一部分代表主机。比如，IP地址172.16.254.1，这是一个32位的地址，假定它的网络部分是前24位（172.16.254），那么主机部分就是后8位（最后的那个1）。处于同一个子网络的电脑，它们IP地址的网络部分必定是相同的，也就是说172.16.254.2应该与172.16.254.1处在同一个子网络。

但是，问题在于单单从IP地址，我们无法判断网络部分。还是以172.16.254.1为例，它的网络部分，到底是前24位，还是前16位，甚至前28位，从IP地址上是看不出来的。

那么，怎样才能从IP地址，判断两台计算机是否属于同一个子网络呢？这就要用到另一个参数**子网掩码（subnet mask）**。

所谓"子网掩码"，就是表示子网络特征的一个参数。它在形式上等同于IP地址，也是一个32位二进制数字，它的网络部分全部为1，主机部分全部为0。比如，IP地址172.16.254.1，如果已知网络部分是前24位，主机部分是后8位，那么子网络掩码就是11111111.11111111.11111111.00000000，写成十进制就是255.255.255.0。

知道"子网掩码"，我们就能判断，任意两个IP地址是否处在同一个子网络。方法是将两个IP地址与子网掩码分别进行AND运算（两个数位都为1，运算结果为1，否则为0），然后比较结果是否相同，如果是的话，就表明它们在同一个子网络中，否则就不是。这里网络是分级的，不会出现错误的检测，等到我以后对于这一块理解的更加透彻了，会再专门写一篇博客进行说明。这里只需要理解到这里就足够了。

比如，已知IP地址172.16.254.1和172.16.254.233的子网掩码都是255.255.255.0，请问它们是否在同一个子网络？两者与子网掩码分别进行AND运算，结果都是172.16.254.0，因此它们在同一个子网络。

总结一下，**IP协议的作用主要有两个，一个是为每一台计算机分配IP地址，另一个是确定哪些地址在同一个子网络。**

有了网络层、链路层、物理层，我们已经可以实现世界上接入互联网上任意两台机器的互联
互通了。
### 4.3 IP数据包
根据IP协议发送的数据，就叫做IP数据包。不难想象，其中必定会包含IP地址信息。前文说过以太网数据包的格式，而IP数据包正是占据了以太网数据包的数据部分。

这充分体现了互联网分层的优势：上层的变动完全不会影响下层的结构。比如，数据链路层使用了以太网协议，网络层即可以使用 IP 协议，即把 IP 数据包塞入以太网数据包的数据部分，也可以使用其他种类的网络层协议替换 IP 协议，即将其他种类的网络层协议数据包塞入以太网数据包的数据部分。

具体来说，IP 数据包也分为包头与数据两个部分。
![ip1]({{site.baseurl}}/assets/img/network/ip1.png)

包头部分主要包括版本、长度、IP 地址等信息，数据部分则是 IP 数据包的具体内容。将 IP 数据包放入以太网数据包后，以太网数据包就变成了如下这样。

![int1]({{site.baseurl}}/assets/img/network/int1.png)

IP 数据包的包头部分的长度为20到60字节，整个数据包的总长度最大为65535字节。因此，理论上，一个 IP 数据包的数据部分，最长为65515字节。前面讲过，以太网数据包的数据部分，最长只有1500字节。因此，如果 IP 数据包超过了1500字节，它就需要分割成几个以太网数据包，分开发送了。

### 4.4 APR 协议
讲到这里，我们知道，如果想要实现两台计算机的通信，我们必须同时知道两个地址，一个是对方的 MAC 地址，另一个是对方的 IP 地址。通常情况下，对方的 IP 地址是已知的（一般通过 DNS 协议获得），而 MAC 地址，我们一般通过 APR 协议获得。

其实这里是分两种情况的，一种情况是，两台机器不在同一个子网络，这时是无法得到对方的 MAC 地址的，只能把数据包发送给网关，让网关去处理。

另一种情况，两台计算机处于同一个子网络，那么我们就可以使用 APR 协议了，得到对方的 MAC 地址。

APR 协议（基于 IP 协议）是这么做的，发送一个数据包（包含在以太网数据包中），其中包含它所要查询的主机的 IP 地址，在对方的 MAC 地址一栏填`FF：FF：FF：FF：FF：FF`，这表示一个广播地址。即子网络的每个主机收到这个包都要这样做：从中取出 IP 地址，与自身的 IP 地址进行比对。如果两者相同，做出回应，向对方报告自己的 MAC 地址，否则丢弃这个包。

有了 APR协议之后，我们就可以查询得到子网内任意机器的 MAC 地址了。一般情况下只有第一次通信时才会需要查询，以后会将用过的 MAC 地址缓冲下来，下次直接本地查找缓冲，没有才会广播查询。

### 4.5 小节
有了网络层、链路层、物理层，我们已经可以实现世界上接入互联网上任意两台机器的互联互通了。

## 五、传输层
### 5.1 传输层的由来
由上节我们知道，有了网络层、链路层、物理层，我们已经可以实现世界上接入互联网上任意两台机器的互联互通了。但是，同一台主机上有很多程序都需要用到网络，比如，你一边浏览网页，一边与朋友在线聊天。当收到一个数据包时，你需要判断它表示的是网页的内容还是在线聊天的内容。

于是，传输层诞生了。**网络层实现的是主机与主机之间的通信，传输层实现的是程序与程序间的交流。**

在传输层中，表示一个数据包到底供哪个程序使用的参数叫做**端口（port）**。它的本质更像是使用网卡的程序的编号。每个数据包都发送到主机的特定端口，所以不同程序可以取走自己所需要的数据。

端口是0到65535之间的一个整数，刚好16个二进制位。其中0到1023端口被系统占用，用户只能选择大于1023的端口。

网络层定义主机位置，传输层定义端口位置。确定了主机与端口，便能实现程序之间的交流。因此，在 Unix 系统中，把`主机+端口`叫做**套接字（socket）**。有了它，便可以进行网络应用程序的开发了。

下面讲讲两个传输层协议，UDP 协议与 TCP 协议。

### 5.2 UDP 协议
UDP 协议就是在数据包中加入了端口信息，它也是标头+数据的格式，如下：
![udp]({{site.baseurl}}/assets/img/network/udp.png)

其中， Head 部分主要定义了发出端口和接收端口， Data 部分是具体的内容。将整个 UDP 数据包放入 IP 数据包的数据部分，结合前面说的，IP 数据包放进以太网数据包的数据部分，最后整个以太网数据包是这个样子的：
![int2]({{site.baseurl}}/assets/img/network/int2.png)

UDP 数据包十分简单， Head 部分一共只有8个字节，总长度不超过65535字节。

### 5.3 TCP 协议
UDP 协议简单有效，容易实现，但是可靠性差，因为一旦它的数据包发出，无法确定对方能否收到。

为了解决这个问题，提高网络可靠性， TCP 协议诞生了。这个协议十分复杂，此处可以简单的认为，它是有确认机制的 UDP 协议，即它的每一个数据包发出后，都会收到一个 ACK 响应包。如果一个数据包丢失，就收不到 ACK，发出方就知道有必要重新发送这个数据包了。

因此，TCP协议能够确保数据不会遗失。它的缺点是过程复杂、实现困难、消耗较多的资源。

TCP 数据包与 UDP 数据包一样内嵌在 IP 数据包的 Data 部分。 TCP 数据包没有长度限制，理论上可以无限长，但是为了保证网络的效率，通常 TCP 数据包的长度不会超过 IP 数据包的长度，以确保单个 TCP 数据包不必再分割。

## 六、应用层
上节中提到，传输层可以使得应用程序之间进行数据传输。这时，我电脑上的应用程序接收到了数据包，并逐层去掉包头，最后只剩下一个数据部分。**应用层的作用就是规定应用程序的数据格式。使得客户端可以将收到的数据包转换成自己想要的格式，并呈现给用户。**

举例来说，通过 TCP/IP 协议，各种各样的程序可以传递数据，比如 Email、WWW、FTP等等。应用层的各种程序协议便规定了电子邮件、网页、 FTP 数据的格式，这些应用程序共同构成了应用层。

应用层是最高的一层，直接面向用户。它的数据就放在 TCP 数据包的数据部分。因此，最终的以太网数据包是如下格式的。

![int3]({{site.baseurl}}/assets/img/network/int3.png)

## 七、总结
上面就是对互联网整个分层结构的解析，总结一下，一台主机在发出一个数据包时需要经历一下步骤：
1. 应用层将数据信息按照标准的格式（如http报文的格式）组织好，然后向下给到传输层;
2. 传输层为应用层的数据包添加包头（比如UDP就是端口信息），变成 UDP 数据包格式，向下给到网络层;
3. 网络层为传输层的数据包添加包头（ip地址等信息），组装成 IP 数据包的格式，向下给到链路层;
4. 链路层为网络层的数据包添加包头（MAC地址等信息），组装成 以太网数据包的格式，向下给到物理层;
5. 物理层将以太网数据包转换成0和1电信号，发出去。

