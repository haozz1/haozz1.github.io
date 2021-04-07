---
title: oracle技巧
date: 2019-07-23 11:17:03
tags: oracle
---


#### 1. python3 中的cx_Oracle

cx_Oracle 模块是没法直接通过pip install的,要想使用,需要配置好几个东西,此处默认已经安装了python3(离线的话需要配置好yum和pip源)

##### 1. oracle instantclient

[oracle官网下载地址](https://www.oracle.com/technetwork/cn/topics/linuxx86-64soft-095635-zhs.html)
官网下载需要注册,我把我用到的 instantclient-basic-linux.x64-12.1.0.2.0.zip [放到了网盘可供下载]()
将zip包放置到/opt/下并解压

##### 2. cx_Oracle 

[下载地址](https://pypi.org/project/cx-Oracle/),下载如图压缩包

{% asset_img cx.jpg %}

下载完成后,解压并进入,然后 python36 setup.py install 等待完成

##### 3. 环境变量

配置环境变量 `vim /etc/profile`
再文件最后加上两行内容
```bash
export ORACLE_HOME=/opt/instantclient_12_1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME
```

如果当前用户进入python3后依然无法 import cx_Oracle,请在shell端在执行一下export操作

```bash
Haozz:~ haozz$ export ORACLE_HOME=/opt/instantclient_12_1
Haozz:~ haozz$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME
```

#### 2. merge into

在oracle中,我们经常会遇到一种情形,需要将B表中的数据插入A表的同时考虑数据是否在A表中存在,如果不存在则插入,存在则更新那条记录的某个字段值.
此时使用`merge into`是最好不过的了(mysql中是 `replace into`)

merge into的实例用法:假如我要把B表中的数据(姓名,身份证号码)插入 购房记录 表中,如果购房记录表中存在这个身份证号码则将标签字段修改为'多次',如果没有则直接插入标签字段为'一次'
在merge into时有一个前提,源表中的身份证号码字段唯一,否则会报错 `ORA-30926`,可见[我的文章 oracle异常中的介绍](http://besthao.cn/2018/12/19/oracle/),这里不再解释

```sql
merge into 购房记录 A using (
    select 
    B.name,
    B.idno 
    from
    购房人员 B
)C on (A.idno=C.idno)
when matched then 
update set A.label='多次' 
when not matched then 
insert (
    A.name,
    A.idno,
    A.label
)values(
    B.name,
    B.idno,
    '一次'
)
```


