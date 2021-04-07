---
title: mac下的u盘刻录
date: 2018-12-11 15:26:29
tags: [Mac, Linux]
---

最近工作中需要将一份超大数据导入到新的机器上,那台机器重新添加了磁盘之后需要重新安装系统,于是新工作又来了.
下载好了CenOS-7-everything.iso后需要做u盘刻录然后给机器安装新系统


##### 1. 查看u盘盘符
`diskutil list` 查看盘符
```bash
sh-3.2# diskutil list
/dev/disk0 (internal):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                         251.0 GB   disk0
   1:                        EFI EFI                     314.6 MB   disk0s1
   2:                 Apple_APFS Container disk1         250.7 GB   disk0s2

/dev/disk1 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +250.7 GB   disk1
                                 Physical Store disk0s2
   1:                APFS Volume Macintosh HD            81.0 GB    disk1s1
   2:                APFS Volume Preboot                 66.2 MB    disk1s2
   3:                APFS Volume Recovery                1.0 GB     disk1s3
   4:                APFS Volume VM                      4.3 GB     disk1s4

/dev/disk3 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *62.1 GB    disk3
   1:             Windows_FAT_32 KINGSTON                61.9 GB    disk3s1
```
可以看到我使用的kingston u盘的卷标是 `disk3s1` 完整路径是`/dev/disk3s1` 

##### 2. 取消挂载
`diskutil umount /dev/卷标`
上面一步知道了我的u盘的路径 所以完整命令为`diskutil umount /dev/disk3s1`    
```bash
sh-3.2# diskutil umount /dev/disk3s1 
Volume KINGSTON on disk3s1 unmounted
```
成功后会得到提示.

##### 3. 写入u盘
`sudo dd if=镜像路径 of=/dev/r卷标 bs=1m` `加r会让写入速度更快,bs为一次写入容量`
结合我的得到完整命令为
`sudo dd if=/Users/dev/Downloads/CentOS-7-x86_64-DVD-1810.iso of=/dev/rdisk3s1 bs=1m`

```bash
sh-3.2# sudo dd if=/Users/dev/Downloads/CentOS-7-x86_64-DVD-1810.iso of=/dev/rdisk3s1 bs=1m
4376+0 records in
4376+0 records out
4588568576 bytes transferred in 143.529066 secs (31969612 bytes/sec)
```
成功后会得到提示
也可以使用`iostat`命令查看磁盘写入状态
例如`iostat -w 2`

##### 4. 弹出u盘
`diskutil eject /dev/卷标`

```bash
sh-3.2# diskutil eject /dev/disk3s1
Disk /dev/disk3s1 ejected
```

然后就可以拔出u盘,拿到新机器上u盘启动安装新系统了.