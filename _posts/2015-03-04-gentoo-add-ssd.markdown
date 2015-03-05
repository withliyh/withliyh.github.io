---
layout: post
title:  "Gentoo 新增 SSD 并切换根分区到 SSD"
date:   2015-03-04 23:16:02
---
#**写在前面**

一直对 SSD 的性能垂涎三尺，加上老 HDD 读起数据来咔咔响让人烦躁，终于一咬牙
啃了一个月的馒头入了`三星850 EVO 250G`。货一到就迫不及待的把 `Gentoo` 转移到 SSD 上

#**对 SSD 进行分区**


分区划分如下

```bash
$ lsblk -o NAME,MOUNTPOINT,FSTYPE,SIZE /dev/sdb
NAME   MOUNTPOINT FSTYPE   SIZE
sdb                      232.9G
├─sdb1 /boot      ext4       1G
├─sdb2 /          ext4      50G
├─sdb3 /home      ext4     180G
└─sdb4 [SWAP]     swap     1.9G
```

我不准备系统待机或休眠，交换分区只分了 1.9G 聊胜于无，如果需要待机或休眠据说交换分区要有内存大小,
为了足够转储内存中的数据

#**转移系统文件**

首先把原系统备份，使用脚本打包成一个jar包
接着还原这个jar包到新分区上

启用交换分区

`mkswap /dev/sdb4`
`swapon /dev/sdb4`

#**优化缓存和临时目录**

设置 portage 编译目录到 /tmp
编辑 /etc/portage/make.conf
>PORTAGE_TMPDIR=”/tmp”

设置 XDG cache 到 /tmp, 如此这般后chrome、firefox的缓存就转移到/tmp去
编辑 /etc/env.d/30xdg-data-local
>XDG_CACHE_HOME=”/tmp/.cache”

执行命令 `env-update`
换成 SSD 后的开机时间太让人惊叹了！
![boot][bootplot]

[bootplot]:     /public/img/bootplot.svg
