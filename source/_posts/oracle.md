---
title: oracle异常
date: 2018-12-19 18:32:11
tags: oracle
---

##### 1. 用户session

意外终端sql或者sqlplus终端关掉可能会使正在执行的session被oracle锁掉
```sql
SQL Error: ORA-00054: resource busy and acquire with NOWAIT specified or timeout expired
00054. 00000 -  "resource busy and acquire with NOWAIT specified"
*Cause:    Resource interested is busy.
*Action:   Retry if necessary.
```
session 被lock了
```sql
SQL> select session_id from v$locked_object;
```
得到sid 比如结果是142
```sql
SQL> SELECT sid, serial#, username, osuser FROM v$session where sid = 142;
```
得到结果比如 142 38 SCOTT oracle
```sql
SQL> ALTER SYSTEM KILL SESSION '142,38';
System altered
```
这样就把lock的session杀掉了

但有时候第三步alter的时候可能说用户会话不存在,这就很麻烦了
处理办法
```sql
select sid, serial#,paddr, status,type, last_call_et,blocking_Session,event,wait_class,state, command from v$session where sid=142
```
得到paddr比如000000015F4DBEA0
```sql
select addr,pid,spid,username,pga_used_mem from v$process where addr = '000000015F4DBEA0'
```
得到spid比如1783
在系统终端直接
```bash
kill -9 1783
```

##### 2. exp导出

> EXP-00003:未找到段(0,0)的存储定义

此表为空表没数据,使用exp无法导出,但不影响导出任务,导入也不会有这个表

> EXP-00006:内部出现不一致性错误

此表为11g分区表间隔分区表,使用exp无法导出,可以将错误表剔除再导出或者使用数据泵expdp

##### 3. imp导入

> ORA-01917 IMP-00017 由于oracle错误,以下语句失败 “GRANT SELECT ON "LVKE" TO "QCZSJ"”

由于一些额外的用户、权限不存在引起的导入错误,不影响导入任务,因为这些与所需数据无关无需导入,不存在才会报错,属正常可以无视

> ORA-00959 表空间xxx不存在

创建user的同时要创建表空间,并为表空间指定表空间文件
```sql
create tablespace xxods logging datafile '/home/oracle/dbf/xx.dbf' size 32m autoextend on next 32m maxsize unlimited extent management local; 
```
也可以顺便将表空间赋予给user
```sql
alter user xxx default tablespace xxx;
```
有时候(SHINYUE40_SDRZ的那些表所属有两个表空间,shine40_sdrz和shineyue40,需要创建两个表空间,不必须赋予给user)会遇到数据需要多个表空间,至于需要的除了同名user的之外还有什么,只能在导入过程中遇到异常才知道,或者就是在导出的时候查看一下导出数据有那些表空间并记录好,在导入之前做好这些准备工作

> ORA-01658 无法为表空间中的xxx段创建INITIAL区

表空间不足,为报错的user的表空间增加表空间文件
```SQL
alter tablespace xxods add datafile '/home/oracle/dbf/xx2.dbf' size 500m autoextend on next 100m maxsize unlimited;
```

> ORA-04098: trigger is invalid and failed re-validation tips 触发器无效或未通过重新验证

同一张表有多个dmp文件,在导入成功第一个dmp文件之后会为这张表创建了触发器,再导入其他dmp文件就会遇到这个错误
得到这张表的触发器(使用PLSQL、NAVICAT或者sql查询),删除这个触发器,并在导入其他dmp文件的时候加上`data_only=y`防止再创建触发器而使得后续再次报错


##### 4. impdp导入

> ORA-39083:对象INDEX无法创建错误  ORA-14102:只能指定一个LOGGING或NOLOGGING子句

引发这一错误的原因是oracle的bug,在创建约束或索引的语句中出现了logging、nologging关键字,解决方法有两种
1. 打补丁
下载 Patch 8795792(oracle官方的下载途径太麻烦,去网上找一找),按照步骤做好
2. 参数加语句
`TRANSFORM=SEGMENT_ATTRIBUTES:N:INDEX TRANSFORM=SEGMENT_ATTRIBUTES:N:CONSTRAINT`
但是我们的导入过程中遇到的并不是跟这个一模一样的bug,报错时查看错误日志中提示的失败SQL,我们提示的是建表语句的错误(原因未知)
解决方法:得到错误表的建表语句,手动在oracle中创建该表,然后单独导入该表(tables=xxx)并加上参数`table_exists_action=truncate`

##### 5. ORA-28000 The account is locked & ORA-28001 The password has expired

oracle11g中默认在default概要文件中设置了 “FAILED_LOGIN_ATTEMPTS=10次”,当输入密码错误次数达到设置值将导致此错误,该用户会自动锁住
我出现这个问题是因为用户密码超时了,尝试次数超过10此被lock
解决办法:
```sql
sqlplus / as sysdba
1>select username,account_status from dba_users where username='TEST';

USERNAME                       ACCOUNT_STATUS
------------------------------ --------------------------------
TEST                          EXPIRED & LOCKED(TIMED)

#可知TEST账户密码过期并且被锁了
#解锁
2>alter user TEST account unlock;

#设置密码检测为无期限
3>select * from dba_profiles where profile='DEFAULT' and RESOURCE_NAME='PASSWORD_LIFE_TIME';

PROFILE                        RESOURCE_NAME                    RESOURCE_TYPE     LIMIT
------------------------------ -------------------------------- --------------    ------------
DEFAULT                        PASSWORD_LIFE_TIME               PASSWORD          180

4>alter profile default limit password_life_time unlimited;
#在查询第三条SQL就会发现结果已经变为了了unlimited
```


##### 6. ORA-30926 Unable to get a stable set of rows in the source tables

Unable to get a stable set of rows in the source tables:无法在源表中获得一组稳定的行,这个错误会出现在使用 `merge into` 时

这个错误是由于数据来源表(即语句中,using后面的from关键字后面的表,这里叫做table_source源表)存在数据重复造成的

> merge into的内部处理是将table_source的每一条记录和table_target的每一条记录对比匹配,匹配到符合条件的记录就会进行修改,匹配不到的话就会insert。如果table_source的匹配列中有重复值的话,等到第二次重复的列值匹配的时候,就会将第一次的update后的值再一次update,就是说合并后的table_target中会丢失在table_source中的记录！！！如果记录丢失的话,两表合并的就没有意义了

所以在进行merge into的时候要保证数据一致(on的字段)

一个简单的处理方法:不考虑数据完整性,只为完成merge into测试,将重复字段进行group by保留其他字段的min()值

```sql
select 
min(xxx1),
min(xxx2),
trim(column_name)
from table_name
group by trim(column_name)
```