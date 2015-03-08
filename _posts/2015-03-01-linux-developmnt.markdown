---
layout: post
title:  "Linux 开发环境和 Mac 一样好用"
date:   2015-03-01 14:34:57
tags: 优雅的开发环境
---

   ![P67][P67IMG]

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

一个稳定而完美的 `Gentoo` 系统

# 内核配置

首先把硬件驱动起来

```
00:00.0 Host bridge: Intel Corporation Xeon E3-1200 v2/Ivy Bridge DRAM Controller (rev 09)
00:01.0 PCI bridge: Intel Corporation Xeon E3-1200 v2/3rd Gen Core processor PCI Express Root Port (rev 09)
00:16.0 Communication controller: Intel Corporation 6 Series/C200 Series Chipset Family MEI Controller #1 (rev 04)
00:19.0 Ethernet controller: Intel Corporation 82579V Gigabit Network Connection (rev 05)
00:1a.0 USB controller: Intel Corporation 6 Series/C200 Series Chipset Family USB Enhanced Host Controller #2 (rev 05)
00:1b.0 Audio device: Intel Corporation 6 Series/C200 Series Chipset Family High Definition Audio Controller (rev 05)
00:1c.0 PCI bridge: Intel Corporation 6 Series/C200 Series Chipset Family PCI Express Root Port 1 (rev b5)
00:1c.1 PCI bridge: Intel Corporation 6 Series/C200 Series Chipset Family PCI Express Root Port 2 (rev b5)
00:1c.2 PCI bridge: Intel Corporation 6 Series/C200 Series Chipset Family PCI Express Root Port 3 (rev b5)
00:1c.3 PCI bridge: Intel Corporation 6 Series/C200 Series Chipset Family PCI Express Root Port 4 (rev b5)
00:1c.4 PCI bridge: Intel Corporation 6 Series/C200 Series Chipset Family PCI Express Root Port 5 (rev b5)
00:1c.6 PCI bridge: Intel Corporation 82801 PCI Bridge (rev b5)
00:1c.7 PCI bridge: Intel Corporation 6 Series/C200 Series Chipset Family PCI Express Root Port 8 (rev b5)
00:1d.0 USB controller: Intel Corporation 6 Series/C200 Series Chipset Family USB Enhanced Host Controller #1 (rev 05)
00:1f.0 ISA bridge: Intel Corporation P67 Express Chipset Family LPC Controller (rev 05)
00:1f.2 SATA controller: Intel Corporation 6 Series/C200 Series Chipset Family SATA AHCI Controller (rev 05)
00:1f.3 SMBus: Intel Corporation 6 Series/C200 Series Chipset Family SMBus Controller (rev 05)
01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Barts PRO [Radeon HD 6850]
01:00.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] Barts HDMI Audio [Radeon HD 6800 Series]
03:00.0 USB controller: NEC Corporation uPD720200 USB 3.0 Host Controller (rev 04)
05:00.0 SATA controller: JMicron Technology Corp. JMB362 SATA Controller (rev 10)
06:00.0 USB controller: NEC Corporation uPD720200 USB 3.0 Host Controller (rev 04)
07:00.0 PCI bridge: ASMedia Technology Inc. ASM1083/1085 PCIe to PCI Bridge (rev 01)
08:01.0 FireWire (IEEE 1394): VIA Technologies, Inc. VT6306/7/8 [Fire II(M)] IEEE 1394 OHCI Controller (rev c0)
09:00.0 SATA controller: Marvell Technology Group Ltd. 88SE9172 SATA 6Gb/s Controller (rev 11)

```



# 桌面环境

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

