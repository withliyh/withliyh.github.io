---
layout: post
title:  "Linux 开发环境和 Mac 一样好用"
date:   2015-03-01 14:34:57
tags: 优雅的开发环境
---

## 我们的目标是

> 一个稳定而完美的 `Gentoo` 系统

## 明确需求

首先要保持系统的整洁，不能有多余的包，其次要满足需求，需要用到的软件要能信手拈来。

声卡、显卡要能正常驱动，同时保持内核尽量短小，不用的功能不编译进内核，不常用的功能编译为模块

在这个前提下，桌面要漂亮，操作简单，有良好的交互。

在使用过程中也会遇到不适应或不顺心的地方，需要慢慢调整，同时记录在案，最终达到我们的目标：一个完美而稳定的 Gentoo 系统。

<table border=”1”>
    <tr>
        <th>需要</th>
        <th>不需要</th>
    </tr>
    <tr>
        <td>经常长期运行</td>
        <td>不待机或偶尔待机</td>
    </tr>
    <tr>
        <td>使用 vpn</td>
        <td>不用 ipv6</td>
    </tr>
    <tr>
        <td>使用虚拟机</td>
        <td></td>
    </tr>
    <tr>
        <td>偶尔听音乐看电影</td>
        <td></td>
    </tr>
    <tr>
        <td>整洁清新的图形界面</td>
        <td></td>
    </td>
    <tr>
        <td>SSH 远程登录</td>
        <td></td>
    </tr>
    <tr>
        <td>Anddoid 开发</td>
        <td></td>
    </tr>
    <td>
        <tr>可能会用到NFS 服务</tr>
        <tr></tr>
    </td>
    <td>
        <tr>NTFS, VFAT文件系统访问</tr>
        <tr></tr>
    </td>
</table>



## 定制系统

这是 12 年的配置，现在有更好的硬件，但还不算过时，在升级到 128G 的 SSD 后，更是焕发了第二春





# 硬件配置
>
> 主板：[SABERTOOTH_P67][P67]
>    
> CPU : [Intel® Xeon® CPU E3-1230 v2(8M Cache, 3.30 GHz)][CPU]
>    
> 内存: [海盗船(CORSAIR)复仇者 DDR3 1600 4GB * 2 台式机内存][MEM]
>   
> 硬盘：2T
>   
> 显卡：[MSI R6850 Hawk][VGA]


   ![P67][P67IMG]

# 内核配置

每个人需求不同，只记录几处重要的地方

> 内核博大精深，不懂的地方太多，以下只是多次尝试后，现在比较好用的配置
>>>
    File systems --->
        <M> FUSE (Filesystem in Userspace) support
        DOS/FAT/NT --->
            <*> MSDOS fs support
            <*> VFAT (Window-95) for FAT
            (936) Default codepage for FAT
            (utf8) Default iocharset for FAT
            <*> NTFS file system support
            [ ] NTFS debugging support
            [*] NTFS write support
        Native language support --->
            (utf8) Default NLS Option
            <*> Simplified Chinese charset (CP936, GB2312)
            <*> ASCII (United States)
            <*> NLS ISO 8859-1 (Latin 1; Western European Languages)
            <*> NLS UTF-8
    Virtualization --->
        <M> Kernel-based Virutal Machine (KVM) support
        <M> KVM for Intel processors support
        <M> Host kernel accelerator for virtio net
    Device Drivers --->
        Graphics support --->
        <M> /dev/agpgart (AGP Support) --->
            <M>Intel 440LX/BX/GX, I8xx and E7x05 chipset support
        Direct Rendering Manager  --->
            < > Direct Rendering Manager (XFree86 4.1.0 and higher DRI support)
        Frame buffer Devices --->
            < > Support for frame buffer devices  ----
        Generic Driver Options --->
            [*] Support for uevent helper
            [*]   Include in-kernel firmware blobs in kernel binary
            (radeon/BTC_rlc.bin radeon/BARTS_mc.bin radeon/BARTS_me.bin radeon/BARTS_pfp.bin radeon/BARTS_smc.bin radeon/SUMO_uvd.bin)
            (/lib/firmware) Firmware blobs root directory
>>>

附上完整的[内核配置文件][KERNEL]



## 桌面环境

首先说一下
Gnome 3.12

# 常用软件

开发工具: `Eclipse` 和 `Android Studio`

截图工具: gnome-screenshot

文本编辑器: sublime 2 gedit vim

图片浏览器: EyeOfGnome(EOG, gnome 默认图片浏览器)

网络浏览器: Chrome Firefox

虚拟机: Boxes

取色器: gcolor2

翻译工具: openyoudao

输入法: fcitx sogoupinyin


[P67]:         http://www.asus.com.cn/Motherboards/SABERTOOTH_P67/
[CPU]:         http://ark.intel.com/zh-cn/products/52271/Intel-Xeon-Processor-E3-1230-8M-Cache-3_20-GHz?_ga=1.248915271.1320732358.1425372536

[P67IMG]:      http://www.asus.com.cn/websites/global/products/ZYgjt71bzlh62Zk9/product_overview.jpg
[MEM]:         http://item.jd.com/615822.html#none
[VGA]:         http://www.chiphell.com/article-756-1.html 
[KERNEL]:      /assets/config/.config
