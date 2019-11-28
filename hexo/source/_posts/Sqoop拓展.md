---
title: Sqoop拓展
date: 2019-04-11 14:34:33
tags: Sqoop
---

sqoop import:
-----
在sqoop import语句中指定 --target-dir参数 后面提供hdfs的目标路径,sqoop会把db数据抽取出来导入到这个路径下.
比如 `-target-dir /user/hive/hivetmpdata`
所以数据会直接导入到这个hivetmpdata下面
	
sqoop job:
-----
sqoop job相当于是可以自动维护数据更新状态的sqoop import,也需要指定`--target-dir`,目的是相同的,只不过它与sqoop import的流程是不一样的

sqoop job在把db数据抽取出来导入到`--target-dir`的过程中会在指定的`--target-dir`父目录下创建一个_sqoop中间目录并把数据存放在此目录下,当sqoop job执行完成后再将数据完整拷贝到--target-dir下.

测试:
-----
使用sqoop job进行导入:
/user/hive/_sqoop 下面开始是没有文件的
{% asset_img 1.png %}
然后开始执行导入任务,当日志打印显示开始执行sqoop job任务的时候再次观察
{% asset_img 2.png %}
这时候产生了临时文件
{% asset_img 3.png %}
当sqoop job 任务执行完毕,这里的中间文件又消失了，而在/user/hive/hivetmpdata/下面多了文件
/user/hive/hivetmpdata/xxxx.xxx

然后测试中止sqoop job
还是开始导入任务，然后在日志打印到开始sqoop job的时候手动停止
{% asset_img 4.png %}

观察_sqoop目录
{% asset_img 5.png %}

    可以看到sqoop 的导入已经是成功的了，并且生成了数据文件，sqoop job中指定了 -m 1 生成了一份数据文件
    因为手动停止了sqoop job任务，所以sqoop后续的操作(将文件拷贝到--target-dir下并重命名、删除临时目录及文件)还没完成，只是完成了oracle到hdfs的操作
大概流程如下:
{% asset_img 6.png %}
对于sqoop job，开发只需要指定--target-dir就可以了，中间目录的创建，写临时文件、拷贝到目标路径并重命名、删除中间目录和临时文件等操作是sqoop内部做的，无需指定参数之类的东西

    任务启动之后可以监控文件夹目录大小变化来知晓任务的状态
    Sqoop import 监控目标目录/user/hive/hivetmpdata/相应表名 文件夹的大小变化
    Sqoop job可以监控/user/hive/_sqoop/相应表名 文件夹的大小变化
