# 搜狗输入法 ubuntu16.04
1. 官网下载`sogoupinyin_2.2.0.0108_amd64.deb`

2. 一般直接安装不能装上`sudo dpkg -i sogoupinyin_2.2.0.0108_amd64.deb`，会报错是因为dpkg不能解决一些依赖问题。

3. 使用`sudo apt-get -f install`可以解决依赖问题。

4. 安装完了就可以装上了`sudo dpkg -i sogoupinyin_2.2.0.0108_amd64.deb`

5. 装上重启后，进入`设置/语言支持`，将键盘输入法系统设置为fcitx

6. 还有`fcitx-config-gtk3`，打开输入法配置，输入法一栏应该有两个，一个中文，一个英文，如果少了一个就不能切换语言了。
