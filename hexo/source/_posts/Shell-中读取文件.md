---
title: Shell 中读取文件
date: 2019-05-23 11:13:28
tags: Linux
---

shell 有好几种读取文件内容的方法,就像代码一样不同的办法解决同一种问题
首先我们创建一个有内容的txt文件
```bash
cat test.txt
aaa
bbb
ccc ddd
```

##### 1.管道
管道顾名思义就是将命令像一个管道一样依次执行,上一个命令的结果当做下一个命令的输入

```bash
#! /bin/sh

cat test.txt | while read line
do
    echo $line
done
```
输出如下:
```text
aaa
bbb
ccc ddd
```
但是值得注意的是如果我希望在循环中做一个累加计算,然后在循环结束后得到累加之后的值,使用管道法是不行的
举个例子
```bash
#! /bin/sh
count=0
cat test.txt | while read line
do
    let count++
    echo $count
done
echo $count
```
这时的输出结果是这样的
```text
1
2
3
0
```
在循环中累加的count值在循环结束后又变成了0,原因还没探明(***TODO***)

##### 2.while
除了使用管道的方法外,也可以直接使用while循环来读取文件内容
```bash
#! /bin/sh
while read line
do
echo $line
done  < test.txt
```
输出结果符合我们的预期
```text
aaa
bbb
ccc ddd
```
以上的两个方法都是使用了read
read命令接收标准输入,或其他文件描述符的输入,得到输入后,read命令将数据放入一个标准变量中
利用read读取文件时,每次调用read命令都会读取文件中的"一行"文本
当文件没有可读的行时,read命令将以非零状态退出

##### 3.for
有while循环的地方肯定有我for循环!!
使用变量在file中循环取值,取值的分隔符由`$IFS`确定
而`$IFS`默认是空白(包括:空格,制表符,换行符),也就是说如果使用默认的分隔符`空格`也会被当做一次新的循环
```bash
#! /bin/sh
for line in `cat test.txt`
do
    echo $line
done
```
这次的输出和上几次的就不同了
```text
aaa
bbb
ccc
ddd
```
`$IFS`是系统变量,如果修改掉可能会造成意想不到的结果,所以如有需要可以在shell里临时赋值给`$IFS`
```bash
#! /bin/sh
IFS="\n"
for line in `cat test.txt`
do
    echo $line
done
```
这样一来输出就回到之前的样子了
```text
aaa
bbb
ccc ddd
```