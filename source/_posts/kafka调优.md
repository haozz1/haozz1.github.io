---
title: kafka调优
date: 2019-06-17 16:19:57
tags: kafka
---

参考文章:
[kafka性能调优](https://blog.csdn.net/vegetable_bird_001/article/details/51858915)
[kafka生产者Producer参数设置及参数调优建议](https://blog.csdn.net/shenshouniu/article/details/83515413)


##### 1. 压缩

producer压缩器,目前支持none（不压缩）,gzip,snappy和lz4.

商业环境推荐:
基于公司物联网平台,试验过目前lz4的效果最好.当然2016年8月,FaceBook开源了Ztandard.官网测试:Ztandard压缩率为2.8,snappy为2.091,LZ4 为2.10


修改producer配置
`vim ../config/producer.properties`
```bash
...
# specify the compression codec for all data generated: none, gzip, snappy, lz4, zstd
compression.type=lz4
...
```

##### 2. 数据清除

kafka的数据都是落地到本地文件里的,定期删除日志
`vim ../config/server.properties`

```bash
...
# 启用删除策略,删除后不可恢复
log.cleanup.policy=delete
# 保留一天,默认7天,可以使用其他时间单位:hours,minutes和ms
log.retention.hours=24
# 段文件配置1GB，有利于快速回收磁盘空间，重启kafka加载也会加快(如果文件过小，则文件数量比较多，
# kafka启动时是单线程扫描目录(log.dir)下所有数据文件)
log.segment.bytes=1073741824
```
