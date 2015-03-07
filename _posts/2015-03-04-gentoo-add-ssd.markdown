---
layout: post
title:  "Gentoo 新增 SSD 并切换根分区到 SSD"
date:   2015-03-04 23:16:02
---
#**写在前面**

	一直对 SSD 的性能垂涎三尺，加上老 HDD 读起数据来咔咔响让人烦躁，终于一咬牙啃了一个月的馒头入了`三星850 EVO 250G`。货一到就迫不及待的把 `Gentoo` 转移到 SSD 上

#**对 SSD 进行分区**

直接使用 `fdisk` 进行分区，现在较新的版本已经支持 `4K` 对齐了
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

格式化分区

```bash
#mkfs.ext4 /dev/sdb1
#mkfs.ext4 /dev/sdb2
#mkfs.ext4 /dev/sdb3
#mkswap /dev/sdb4
```

如果以后需要休眠也可以用重新制定交换分区
比如挂到 HDD 上：`swapon /dev/sda8`

#**转移系统文件**

首先把原系统备份，使用脚本打包成一个`tar.gz`包
接着还原这个包到新分区上,网上搜到一个针对 `gentoo` 备份的脚本，修改下增加自己的排除目录，效果很好


脚本最终生成的核心备份命令如下：

```bash
(/usr/bin/find /var/db;
echo /usr/src/linux-3.18.7-gentoo/.config;
echo /var/log/emerge.log;
echo /usr/src;
echo /usr/portage;
echo /tmp;
echo /sys;
echo /proc;
echo /mnt/.keep;
echo /mnt;
echo /home;
echo /dev/console;
echo /dev/null;
/usr/bin/find /bin /boot /dev /etc /home /lib /lib32 /lib64 /lost+found 
/media /mnt /opt /proc /root /run /sbin /sys /tmp /usr /var
-path /dev -prune -o
-path /lost+found -prune -o
-path /mnt -prune -o
-path /proc -prune -o
-path /sys -prune -o
-path /tmp -prune -o
-path /usr/portage -prune -o
-path /usr/src -prune -o
-path /var/log -prune -o
-path /var/tmp -prune -o
-path /var/db -prune -o
-path /var/cache/edb -prune -o
-path /mnt/backups/stage4 -prune -o
-path /home/lost+found -prune -o
-path /home/myuserhome -prune -o
-path /usr/src/linux-3.18.7-gentoo -prune -o
-print)
| /bin/tar --gzip --preserve-permissions --create --absolute-names --totals --ignore-failed-read -v --file /mnt/backups/stage4/blackbox-stage4-2015.03.06-minimal.tar.gz --no-recursion -T -
```
这么复杂的打包脚本，无非是想做到排除指定目录时保留目录结构，并且如果被排除的目录有刚好又想保留的文件可以被包含进去

然后恢复备份文件到 SSD 分区


```bash
#mkdir /mnt/gentoo
#mount /dev/sdb2 /mnt/gentoo
#mkdir /mnt/gentoo/boot
#mount /dev/sdb1 /mnt/gentoo/boot
#tar xzvpf stage4.tar.gz -C /mnt/gentoo/
#mount -t proc none /mnt/gentoo/proc
#mount -o bind /dev /mnt/gentoo/dev
#chroot /mnt/gentoo /bin/bash
#env-update
#source /etc/profile
#exit
```

然后转移 `home` 目录到 `SSD` 分区, 我只想转移配置文件，不转移自己的文件。
大多配置文件都是以 `.` 开头的隐藏文件，直接用 `ls` 命令导出到列表,根据列表排除

```bash
$cd ~/
$ls > ext.txt
#编辑 `ext.txt` 增删想要排除的列表
$tar czvf /tmp/home.tar.gz -T ext.txt .
#mkdir /mnt/gentoo/home/myuserhome
#chown myuserhome:myuserhome /mnt/gentoo/home/myuserhome
$tar xzvf /tmp/home.tar.gz -C /mnt/gentoo/home/myuserhome
```

一定不能忘记修改 `fstab` 文件,编辑 `/mnt/gentoo/etc/fstab`, 我的配置文件如下:
有人说添加 `discard` 选项虽然会延长 SSD 使用手边但会影响删除文件的效率,
现在先这样用着之后再修改周期性调用 `fstrim` 命令，然后据说增加 `noatime` 选项
在编译内核时可以使用硬盘写入数据量减少大约 `13%`,次标志阻止访问文件时更新文件的
最近访问的时间戳，减少写入量是一定有的，有没有 `13%` 这么多就不知道了。

```
/dev/sdb1		/boot		ext4		noatime,discard		0 2
/dev/sdb2		/		ext4		noatime,discard	0 1
/dev/sdb3		/home		ext4		noatime,discard		0 2 
/dev/sdb4		none		swap		sw,discard		0 0
tmpfs			/tmp		tmpfs		noatime,nodiratime,size=6G 0 0
```

接着更新 `grub` 的引导菜单

```bash
#grub2-mkconfig -o /boot/grub/grub.cfg
```

接着重启 `reboot`，开始享受 `SSD` 飞一般的速度吧!

如果想把 `grub` 安装到 `/dev/sdb` 

```bash
grub2-install /dev/sdb
```

#**优化缓存和临时目录**

换到 SSD 可以慢慢研究慢慢优化，当前先把编译的临时目录移出 SSD

设置 portage 编译目录到 /tmp, tmp 已经挂载到内存了，
可能编译 `firefox, chrome` 这种巨头不够用，那就直接使用 bin 包

编辑 `/etc/portage/make.conf`

```
PORTAGE_TMPDIR=”/tmp”
```

设置 XDG cache 到 /tmp, 如此这般后chrome、firefox的缓存就转移到/tmp去
编辑 `/etc/env.d/30xdg-data-local`

```
XDG_CACHE_HOME=”/tmp/.cache”
```

执行命令 `env-update && source /etc/profile` 更新环境变量

换成 SSD 后的开机时间太让人惊叹了！
![boot][bootplot]

[bootplot]:     /assets/img/bootplot.svg
