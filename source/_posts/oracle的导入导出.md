---
title: oracle的导入导出
date: 2018-12-12 18:16:08
tags: oracle
---

oracle 提供了完善的exp/imp功能,可以用来备份数据.
当然,plsql developer等数据库连接工具提供了图形化的操作功能,但是那些功能也是在你选择了任务提交后将你所做的操作转换为系统命令提供给系统的,所以我就简单的总结一下在使用exp/imp过程中的经验.

### 1. 准备

在做数据导出之前,我们先要做一些准备工作,这些工作项你可以选择在后续的过程中做甚至不做,但是为了后续工作能少出现一些意外,或者让结果更尽如人意,还是准备一下为好：)

##### 1.1. 更多功能

exp/imp 是oracle提供的用来导出/导入数据的功能,相似的,oracle还提供了 expdp/impdp 两个功能.
简单来说expdp和impdp是exp/imp的加强版,他们的效率更高,允许数据多样性更高(这个后面会说到),但是相比于exp/imp可以同时在client和server使用来说,expdp/impdp是server端的工具程序,他们只能在oracle server使用.
imp只适用于exp导出的文件,不适用于expdp导出文件;impdp只适用于expdp导出的文件,而不适用于exp导出文件.
对于10g以上的服务器,使用exp通常不能导出0行数据的空表,而此时必须使用expdp导出.

而无论你是用哪一套命令,你所在的操作环境必须拥有oracle.

##### 1.2. 流水线

这里的流水线的意思是 `源oracle->exp client->imp client->目标oracle` 这一串工作的流程.
* 尽量保证流水线上的每一步的字符集是相同的.
而关于oracle字符集的讲解和设置,[这里](http://www.cnblogs.com/rootq/articles/2049324.html)讲的很详细.

* 尽量保证版本一致.
如果你的流水线中 exp client和imp client为单独的两台机器而非源/目标机器(这样的情况属于大多数,因为很多情况你是无法去相应的机器上操作的,这也是没有使用expdp/impdp的原因),尽量保证四个机器的oracle版本一致,我用的都是oracle11.
oracle 的很多版本都提供了单独的工具包,这样你可以不用安装完整的oracle就可以使用其中的exp/imp命令,我刚开始的时候就是用了oracle12中的`instantclient-basic-windows.x64-12.2.0.1.0` 和 `instantclient-tools-windows.x64-12.2.0.1.0`两个包,其中tools就是exp等一些命令的包,但是因为与我的源oracle版本不一致而放弃使用改为安装完整的oracle11,另外一件令我很费解的事是oracle11的tools包中竟然没有exp等命令文件...

##### 1.3. 目标oracle

为你的目标oracle创建好新用户,赋予用户dba权限,如有必要需要同时赋予表空间

### 2. exp 导出

示例命令
`exp test/test@localhost/ee file=/home/oracle/test.dmp log=/home/oracle/test.log`
导出test用户下的所有数据,包括数据库连接、序号等.
`exp jqods/jqods@10.53.188.228/rzods File=f:\dmp\jqods.dmp log=f:\dmp\jqods.log tables=(xxxx1,xxx2)`
全用户导出的话省略tables,缺省full 等同于`full=y` 全用户导出
`exp jqods/jqods@10.53.188.228/rzods File=f:\dmp\jqods.dmp log=f:\dmp\jqods.log`

* 使用缓冲(速度差异很大)
`exp jqods/jqods@10.53.188.228/rzods File=f:\dmp\jqods.dmp log=f:\dmp\jqods.log buffer=4096000`
网上有人说buffer建议在1024000-10240000之间,自己把控吧,我用的40960000

* 使用direct
`exp jqods/jqods@10.53.188.228/rzods File=f:\dmp\jqods.dmp log=f:\dmp\jqods.log direct=y recordlength=65536 `
* 直接导出,数据从磁盘读入到高速缓存,直接写入到最终文件,所以没有数据行检查与匹配的过程,据不权威的评测结果,性能有50%到70%的提升
* recordlength参数(IO缓冲大小),与direct参数配对使用,默认该参数为1024bytes,上面的例子为64K(最大值也为64K)
* 使用direct后,buffer参数失效;
* 使用direct,不支持query子句(没有行匹配的过程),不支持带Blob类型字段的表,但是系统会自动判断、自动切换,也就是说不会因为一张表的问题,导致整个schema不能使用direct备份;
* direct不支持表空间导出

{% asset_img exp.png %}
可以看到使用exp导出时,空表会出现异常,但不会影响整个操作,另外导出整个user同时导出了数据之外的很多东西,如果只要数据可以选择`tables=()`

如果导出中还遇到以下错误,可以尝试解决
{% asset_img yichang.png %}

我还遇到过在导出过程中突然报错`内部出现不一致性错误`
{% asset_img cuowu.png %}
而造成这个错误的原因是oracle11后,数据表的新特性,网上说的都是建议使用数据泵(expdp/impdp),可是因为我无法操作server端,只好用`tables=()`得到这个user的所有表,再把有错误的表删掉.(至于怎么知道是哪个表出现问题,我不会告诉你我是一个表一个表试的,或者你看一下所有表的列表顺序,一般提示错误时还没打印出的那张表就是有问题的表)

### 3. imp 导入

导入过程终于可以用server端了！！
但是因为导出用的exp,导入只能用imp,貌似也没有什么不一样...
`imp tlods/tlods@localhost/orcl fromuser=tlods touser=tlods file=f:dmp\tlods.dmp buffer=4096000`
因为我是按user导出的,所以我直接再按user导入
{% asset_img daoru.png %}
如果这时候又出现了异常,当然选择继续忽视！
{% asset_img shibai.png %}
而如果遇到这个错误,那就得看一下了,因为这样的话这个表是没有导入进去的,分析完就知道了是这个表需要表空间,而我们的目标oracle里是没有的.
现在遇到的情况是如果需要导入的表中存在LOB字段,就会报错 `oracle error 959 `
解决办法:
* 先将这个user导入的数据删除,就是把那些已经成功的表删除 `drop table xxods.xxxx`
* 创建表空间
```sql
create tablespace xxods logging datafile '/home/oracle/dbf/xx.dbf' size 32m autoextend on next 32m maxsize unlimited extent management local;
```
* 为用户赋予表空间
```sql
alter user xxods default tablespace xxods;
```
* 再次导入就可以了

如果过程中又遇到了如下问题`无法为表空间xxx中的段创建INITIAL区`
{% asset_img kongjian.png %}
原因是表空间不足了,创建表空间时指定的文件大小限制为最大32G,为此表空间再添加文件就是了.
```sql
alter tablespace xxods add datafile '/home/oracle/dbf/xx2.dbf' size 500m autoextend on next 100m maxsize unlimited
```

>思考:在exp导出的时候指定表 `tables=()`只导出表数据的话就不会导出一些表空间之类的东西,那么这样再imp的时候还会不会出现这个错误呢？
