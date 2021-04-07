---
title: Hive配置MySQL metastore
date: 2018-09-29 10:15:46
tags: Hive
---

hive中除了保存真正的数据以外还要额外保存用来描述库、表、列的数据,称为hive的元数据.
这些元数据又存放在何处呢？hive需要将这些元数据存放在另外的关系型数据库中.

发行版的Hadoop无论是CDH还是HDP安装Hive之前都可以提前配置好MySQL服务来使用MySQL管理Hive的元数据,通过Cloudera Manager或者Ambari管理Hadoop集群来配置MySQL和Hive.过程这里不做描述.

这里讲的是以前使用社区版Hadoop时单独安装Hadoop集群和Hive需要修改的工作.

----

Hive中除了保存真正的数据以外还要额外保存用来描述库、表、列的数据,称为Hive的元数据.
这些元数据又存放在何处呢？Hive需要将这些元数据存放在另外的关系型数据库中.
如果不修改配置Hive默认使用内置的derby数据库存储元数据.
derby是apache开发的基于java的文件型数据库.
可以检查之前执行命令的目录,会发现其中产生了一个`metastore.db`的文件,这就是derby产生的用来保存元数据的数据库文件.
derby数据库仅仅用来进行测试,真正使用时会有很多限制.最明显的问题是不能支持并发.经测试可以发现,在同一目录下使用无法同时开启hive,不同目录下可以同时开启hive但是会各自产生metastore.db文件造成数据无法共同访问.
所以真正生产环境中我们是不会使用默认的derby数据库保存hive的元数据的.

hive目前支持derby和mysql来存储元数据.

配置hive使用mysql保存元数据信息：
删除hdfs中的`/user/hive`

```bash
hadoop fs -rmr /user/hive
```
	
复制`hive/conf/hive-default.xml.template为hive-site.xml`

```bash
cp hive-default.xml.template hive-site.xml
```
 
在<configuration>中进行配置
			
```xml
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://hadoop01:3306/hive?createDatabaseIfNotExist=true</value>
    <description>JDBC connect string for a JDBC metastore</description>
</property>

<property>	  
	<name>javax.jdo.option.ConnectionDriverName</name>
	<value>com.mysql.jdbc.Driver</value>
	<description>Driver class name for a JDBC metastore</description>
</property>

<property>
	<name>javax.jdo.option.ConnectionUserName</name>
	<value>root</value>
	<description>username to use against metastore database</description>
</property>

<property>
	<name>javax.jdo.option.ConnectionPassword</name>
	<value>root</value>
	<description>password to use against metastore database</description>
</property>

```

* 手动创建hive元数据库,注意此库必须是latin1,否则会出现奇怪问题！所以推荐手动创建！并且创建库之前不能有任意的hive操作,否则自动创建出来的库表将使用mysql默认的字符集,仍然报错！

* 另一种方法是修改mysql的配置文件,让mysql默认编码集就是latin1,这样hive自动创建的元数据库就是latin1的了,但是这已修改将会影响整个mysql数据库,如果mysql中有其他库,这种方式并不好.

```bash
create database hive character set latin1
```

将mysql的连接jar包拷贝到`$HIVE_HOME/lib`目录下

如果出现没有权限的问题,在mysql授权(在安装mysql的机器上执行)
``` bash
mysql -uroot -p

#(执行下面的语句  *.*:所有库下的所有表   %：任何IP地址或主机都可以连接)
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

再进入hive命令行,试着创建库表发现没有问题.
		
测试发现开启多个连接没有问题.

连接mysql,发现多了一个hive库.其中保存有hive的元数据.DBS-数据库的元数据信息,TBLS-表信息.COLUMNS_V2表中字段信息,SDS-表对应hdfs目录