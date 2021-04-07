---
title: 使用SecureCRT传输文件
date: 2018-09-29 14:10:25
tags: [Mac, SecureCRT]
---

Mac中没有Xshell,远程调控服务器只好使用了一个叫SecureCRT的软件.

开发或部署过程中都经常需要把文件上传到服务器或者把服务器上的文件下载到本地,这就需要我们的SecureCRT终端能实现文件传输功能了.

SecureCRT下的文件传输协议有以下几种：ASCII、Xmodem、Ymodem、Zmodem
{% asset_img 3.png %}

* ASCII：这是最快的传输协议,但只能传送文本文件.
* Xmodem：这种古老的传输协议速度较慢,但由于使用了CRC错误侦测方法,传输的准确率可高达99.6%.
* Ymodem：这是Xmodem的改良版,使用了1024位区段传送,速度比Xmodem要快.
* Zmodem：Zmodem采用了串流式（streaming）传输方式,传输速度较快,而且还具有自动改变区段大小和断点续传、快速错误侦测等功能.这是目前最流行的文件传输协议.

使用SecureCRT传输主要可以采用两种方式

### 使用lrzsz
---

lrzsz是一款在linux里可代替ftp上传和下载的程序.
之前使用Xshell的时候用的都是lrzsz,在SecureCRT中也可以正常使用

安装lrzsz
```bash
yum -y install lrzsz # -y:安装过程的提示选择全部为"yes"

```
安装好了lrzsz之后就可以通过命令来直接拖拽文件时间上传或下载了.

* rz:received 输入rz时意味着服务器需要接收文件及你要上传文件到服务器上.
在服务器终端中直接输入rz服务器会进入等待状态,找到需要上传的文件即可.
{% asset_img 4.png %}

{% asset_img 5.png %}
这时候在目录下查看一下就会发现多出了刚刚上传的文件了.

>需要注意的是：上传的时候,如果上传到的linux目录有同名的文件,是无法上传的,需要先删掉linux上的同名文件.

* sz:send	输入sz时意味着服务器要发送文件及你要把服务器上的文件下载下来.
下载一个文件：sz filename 
下载多个文件：sz filename1 filename2
下载dir目录下的所有文件，不包含dir下的文件夹：sz dir/*
在终端输入`sz filename1 filename2 filename3`,回车后出现如下截图所示
{% asset_img 6.png %}

选择好存放目录之后点击确定
{% asset_img 7.png %}
上传完成后相应目录下就多出了在服务器上下载的文件.

* 修改默认位置
文件上传、下载存放的默认位置在securtCRT中设置,位于：
英文版 `options — session options — X/Y/Zmodem`
中文版 `选项— 会话选项— X/Y/Zmodem`

### 使用SFTP
---

SFTP是Secure File Transfer Protocol的缩写,安全文件传送协议.可以为传输文件提供一种安全的网络的加密方法.SFTP 与 FTP有着几乎一样的语法和功能.

在SecureCRT中打开SFTP
{% asset_img 1.png %}

在SFTP窗口中输入命令定位到本地和服务器的目录下
```bash
lpwd #查看本地路径
lcd #定位到本地目录下

pwd #查看服务器路径
cd #定位到服务器路径下
```
{% asset_img 2.png %}
本地定位到需要上传文件所在的目录,服务器上定位到需要上传到的目录下
```bash
put #上传
get #下载
```

```sh
sftp> put tables.txt
Uploading tables.txt to /home/hive/anfang/tables.txt
  100% 420 bytes    420 bytes/s 00:00:00     
/Users/dev/Desktop/tables.txt: 420 bytes transferred in 0 seconds (420 bytes/s)
sftp> 
```

```bash
sftp> get tables.txt
Downloading tables.txt from /home/hive/anfang/tables.txt
  100% 420 bytes    420 bytes/s 00:00:00     
/home/hive/anfang/tables.txt: 420 bytes transferred in 0 seconds (420 bytes/s)
sftp> 
```

