---
title: GitHub/GitLab中的Git命令
date: 2018-09-20 10:34:04
tags: Git
---

无论是使用知名的GitLab、GitHub还是使用国内的码云，Git命令都是绕不过的一道门槛。Git是目前世界上最先进的分布式版本控制系统。我们没有必要成为Git大师，只需要学会Git的一些简单命令以实现对以上几种代码托管平台的使用就好了。下面以使用GitLab为例(各大托管平台操作都很相似甚至一样)


前提：已经安装了Git并在GitLab上配置好了SSH免密而且创建好了远程项目
### 一、将本地项目上传到GitLab上：

##### 1、创建空文件夹(本地仓库)
{% asset_img 1.jpg %}

##### 2、右键点击文件夹选择 Git Bash Here&nbsp;进入Git命令行模式
{% asset_img 2.jpg %}
##### 3、连接远程项目(GitLab中创建好的项目中复制SSH地址)
{% asset_img 3.jpg %}

##### 4、克隆项目到本地
{% asset_img 4.jpg %}
此时本地文件夹中产生了文件
{% asset_img 5.jpg %}

##### 5、在命令行模式下进入此文件夹
{% asset_img 6.jpg %}

##### 6、进入项目后默认为master分支
{% asset_img 7.jpg %}
一般master为生产分支，开发项目时要创建其他分支并在上面操作项目(此处要将本地项目上传到Neo4jGraph分支)查看分支(因为我远程项目已经创建了两个分支，所以此处查看时有三个分支master、Neo4jGraph、intoNeo4j)
```bash
git branch -a #查看全部分支
```
{% asset_img 8.jpg %}
```bash
git checkout xxx #切换到xxx分支
git checkout -b xxx #创建xxx分支并切换
```
##### 7、因为要将项目上传到Neo4jGraph分支，切换到Neo4jGraph分支上 
{% asset_img 9.jpg %}

##### 8、将本地项目加进本地仓库中(graph-platform为本地项目，Readme文件为项目文档，在GitLab查看时会自动展示readme文件)
{% asset_img 10.jpg %}

##### 9、将添加操作加到暂存区git add .&nbsp; &nbsp;将文件夹中的内容全部添加到暂存区(如果需要将内容全部上传的话)
```bash
git add xxx #将xxx添加到暂存区
```
{% asset_img 11.jpg %}

##### 10、查看状态(可知我们的操作已经被添加到了暂存区等待提交)
```bash
git status #查看状态
```
{% asset_img 12.jpg %}

##### 11、把暂存区的内容提交到当前分支(此时已经将项目上传到了本地分支上)
```bash
git commit -m "xxx" #将暂存区内容提交到当前分支并起了个版本名称标识
```
{% asset_img 13.jpg %}

##### 12、将本地分支push到远程
git push &lt;远程主机名&gt; &lt;本地分支名&gt;:&lt;远程分支名&gt;&nbsp;&nbsp; &nbsp;但一般为了方便将远程主机名默认为 origingit 
```bash
push origin Neo4jGraph #将内容push到远程Neo4jGraph分支上
```
没有的话远程自动创建
```bash
git push origin Neo4jGraph:Neo4jGraph
```
{% asset_img 14.jpg %}

##### 13、查看GitLab，项目已经成功上传
{% asset_img 15.jpg %}

>如果要删除项目中的某个文件夹或者文件则只需要进入分支后将相应文件删除然后再将留下的内容全部add再commit最后push就好了
```bash
ls #查看有什么内容
rm -rf xxx #强制并递归删除文件xxx
git add . #将剩余内容再次添加到到暂存区
git commit -m "xxx" #提交
git push #上传
```