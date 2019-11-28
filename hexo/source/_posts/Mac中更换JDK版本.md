---
title: Mac中更换JDK版本
date: 2018-09-21 16:40:38
tags: Mac
---

作为一个程序猿拿到Mac之后的第一步就是开机开始安装开发环境了，想都没想就立马下载最新版本JDK，配置完后就看到文档里要求JDK8了。
果然愣头青没有好下场 ：）
<!--more-->
本着愣头青的原则直接来到Java目录下
`rm -rf`
`java -version` 查一下没有java了 
重新下了JDK8但是无法正常安装
找了下解决办法 解释挺多 大概意思就是电脑中还有一些Java所依赖的文件
##### 解决方法：

* 调出终端窗口，依次输入如下命令:

```bash
sudo rm -fr /Library/Internet\Plug-Ins/JavaAppletPlugin.plugin
sudo rm -rf /Library/PreferencesPanes/JavaControlPanel.prefPane
sudo rm -rf ~/Library/Application\ Support/Java
```
输入第一个命令时会提示输入密码，就是电脑密码，该命令只会卸载Mac自带的JDK版本，一般是1.6。

* 若是要卸载1.6以上的版本，同样的方法：打开终端进入JDK所在的目录`(/Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk)`执行命令：

```bash
rm -rf jdk1.8.0_151.jdk3
```
请勿尝试通过从 /usr/bin 删除 Java 工具来卸载 Java。此目录是系统软件的一部分，下次对操作系统执行更新时，Apple 会重置所有更改。
安装好新的jdk之后 会默认在 /Library下创建Java目录 在**/Library/Java/JavaVirtualMachines** 下有jdk1.8.0_181.jdk
{% asset_img 1.jpg %}
其中Contents下的Home文件夹，是该JDK的根目录。

{% asset_img 2.jpg %}

* 配置环境变量

如果你是第一次配置环境变量，可以使用`touch .bash_profile`创建一个.bash_profile的隐藏配置文件(如果你是为编辑已存在的配置文件，则使用"open -e .bash_profile"命令)：

{% asset_img 3.jpg %}
输入`open -e .bash_profile`命令：
{% asset_img 4.jpg %}

输入如下配置：
```bash
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home
PATH=$JAVA_HOME/bin:$PATH:.
CLASSPATH=$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:.
export JAVA_HOME
export PATH
export CLASSPATH
```
然后保存关闭该窗口。
{% asset_img 5.jpg %}
使用`source .bash_profile`使配置生效输入 `echo $JAVA_HOME` 显示刚才配置的路径这时候就可以 `java -version` 查看新安装的Java版本了
{% asset_img 6.jpg %}

>拥有多个版本JDK时更换版本

* 在.bash_profile文件中配置多条JAVA_HOME 

```bash
export JAVA_6_HOME=/System/Library/Java/JavaVirtualMachines/1.6.0.jdk/Contents/Home
export JAVA_7_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0.jdk/Contents/Home
export JAVA_8_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0.jdk/Contents/Home
 
export JAVA_HOME=$JAVA_6_HOME
```

* alias命令动态切换JAVA_HOME的配置

```bash
alias jdk8='export JAVA_HOME=$JAVA_8_HOME'
alias jdk7='export JAVA_HOME=$JAVA_7_HOME'
alias jdk6='export JAVA_HOME=$JAVA_6_HOME’
```

* 输入完成后保存执行下面命令

重新执行.bash_profile文件
```bash
source ~/.bash_profile
```