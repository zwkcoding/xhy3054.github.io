---
layout: post
title: SSH简要教程
date: 2018-04-16 10:07:24.000000000 +09:00
img: ssh.jpg # Add image post (optional)
tags: [linux]
---


### what's SSH
通俗来讲，**SSH**是一种网络协议，主要用于远程登陆。由于这种登陆中信息是加密的，所以即使被中途拦截，也不会泄露密码。


### SSH基本用法
**username**为你登陆的远程主机用户名（若本地用户名与远程用户名一致，可省略`username@`），**host**为你登陆的远程主机ip，此命令默认使用远程主机的**22**端口进行登陆。
```bash
$ ssh username@host
```

若想指定端口号,可用**p**参数（例如指定**33**端口发送登陆请求）
```bash
$ ssh -p 33 username@host
```


### SSH免密登陆
每次登陆都需要输入密码会比较麻烦，可以设置免密登陆。

免密登陆的原理是：
>用户将自己的公钥存储在远程主机上,在登陆时,远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码。

因此,用户首先需要提供自己的公钥,可以进入目录`~/.ssh`查看一下,`id_rsa.pub`为私钥,`id_rsa`为公钥。如果没有,可以用如下方式生成:
```bash
$ ssh-keygen
```

然后用如下命令,可以将公钥传送给远程主机
```bash
$ ssh-copy-id username@host
```

正常情况至此应该已经大公告成,如果没有,请查看远程主机的`/etc/ssh/sshd_config`文件,将下面几行的注释取消
```
　　RSAAuthentication yes
　　PubkeyAuthentication yes
　　AuthorizedKeysFile .ssh/authorized_keys
```

然后重启远程主机的***SSH***服务(ubuntu)
```bash
$ service ssh restart
```

大功告成!


### 本机与远程主机之间文件传输
由本机往远程主机路径`~/picture/`传输文件**a**
```bash
$ scp a username@host:~/picture/
```

由远程主机往本机路径`~/picture/`传输文件**b**(本机**localhost**可以由`ifconfig`命令查看)
```bash
$ scp b localusername@localhost:~/picture/
```



### SSH登陆知识常识
SSH之所以能够保证安全，原因在于它采用了公钥加密。

整个过程是这样的：
>- 远程主机收到用户的登录请求，把自己的公钥发给用户。
>- 用户使用这个公钥，将登录密码加密后，发送回来。
>- 远程主机用自己的私钥，解密登录密码，如果密码正确，就同意用户登录。

这个过程本身是安全的，但是实施的时候存在一个风险：如果有人截获了登录请求，然后冒充远程主机，将伪造的公钥发给用户，那么用户很难辨别真伪。因为不像https协议，SSH协议的公钥是没有证书中心（CA）公证的，也就是说，都是自己签发的。

可以设想，如果攻击者插在用户与远程主机之间（比如在公共的wifi区域），用伪造的公钥，获取用户的登录密码。再用这个密码登录远程主机，那么SSH的安全机制就荡然无存了。这种风险就是著名的"中间人攻击"（Man-in-the-middle attack）。

其实到目前为止并没有有效阻止“中间人攻击的方法”，因此在用户首次尝试登陆远程主机时，SSH会做出如下提示：
```
$ ssh user@host

　　The authenticity of host 'host (12.18.429.21)' can't be established.

　　RSA key fingerprint is 98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d.

　　Are you sure you want to continue connecting (yes/no)?
```
即无法确定host主机的真实性，只知道它的公钥指纹，如果你可以核对这个公钥指纹，即可确保安全登陆。