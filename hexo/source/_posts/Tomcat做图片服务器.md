---
title: Tomcat做图片服务器
date: 2018-10-24 14:48:52
tags: [Tomcat,Linux]
---

使用Linux做服务器时常常将大量图片存放到某个目录下面
而图片往往是某些数据下的一个属性,这些图片一般都要随着他的父节点一起存入数据库中,而在数据库中表示图片的字段存储的并不是该图片所在的绝对路径,而是一段供浏览器可访问的虚拟路径
这就需要配置图片服务器使得这些图片可被访问

###### 1.安装Tomcat
---
下载并解压安装包,配置好Tomcat依赖的JDK环境变量

###### 2.修改文件
---
在Tomcat根目录下 /conf 下有server.xml文件
修改如下:
```xml
    <Engine name="Catalina" defaultHost="此处将localhost改为自己的公网ip">
 
      <Host name="同上面的defaultHost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
				<Context docBase="本地图片的磁盘存放路径,如 /home/imgs/" path="通过Tomcat访问的路径(虚拟路径),如/imgs/"/>

```
###### 3.访问
在浏览器输入访问
`ip:8080/imgs/xxxx.jpg`
或者修改server.xml文件 修改 HTTP访问端口为80即可在访问地址输入中省略端口访问
`ip/imgs/xxxx.jpg`
