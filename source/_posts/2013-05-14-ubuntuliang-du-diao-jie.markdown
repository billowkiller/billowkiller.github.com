---
layout: post
title: "ubuntu亮度调节"
date: 2013-05-14 01:09
comments: true
categories: [ubuntu]
---

1.\
sudo gedit /etc/X11/xorg.conf 把

Section "Device" 　　　　\
Identifier "Device0" 　　　　\
Driver "nvidia" 　　　　\
VendorName "NVIDIA Corporation" EndSection 改成

Section "Device" 　　　　\
Identifier "Device0" 　　　　\
Driver "nvidia" 　　　　\
VendorName "NVIDIA Corporation" 　　　　\
Option "RegistryDwords" " EnableBrightnessControl=1"\
EndSection

然后保存，退出，重启之后，你 就会发现可以调节屏幕背光亮度了

2.\
网上有很多在Ubuntu
Linux下调节笔记本屏幕亮度的方法，有的调的是亮度但不是背光亮度，有的调背光亮度的方法在我的电脑上不好使……找了半天发现这个方法，适用范围应该比较广（起码在我这里好用）。

首先，进入终端，输入lspci命令，列出各种设备的地址
<!--more-->
www.linxidc.com@Ubuntu:\~\$ lspci\
00:00.0 Host bridge: Intel Corporation Mobile 945GM/PM/GMS, 943/940GML
and 945GT Express Memory Controller Hub (rev 03)

00:02.0 VGA compatible controller: Intel Corporation Mobile 945GM/GMS,
943/940GML Express Integrated Graphics Controller (rev 03)\
00:02.1 Display controller: Intel Corporation Mobile 945GM/GMS/GME,
943/940GML Express Integrated Graphics Controller (rev 03)\
00:1b.0 Audio device: Intel Corporation N10/ICH 7 Family High Definition
Audio Controller (rev 02)\
00:1c.0 PCI bridge: Intel Corporation N10/ICH 7 Family PCI Express Port
1 (rev 02)\
00:1c.1 PCI bridge: Intel Corporation N10/ICH 7 Family PCI Express Port
2 (rev 02)\
......\
发现00:02.0是VGA设备，于是我们修改它的属性

sudo setpci -s 00:02.0 F4.B=FF

解释一下：

setpci是修改设备属性的命令\
-s表示接下来输入的是设备的地址\
00:02.0 VGA设备地址（:.）\
F4 要修改的属性的地址，这里应该表示“亮度”\
.B
修改的长度（B应该是字节（Byte），还有W（应该是Word，两个字节）、L（应该是Long，4个字节））\
=FF 要修改的值（可以改）

我这里00是最暗，FF是最亮，不同的电脑可能不一样。

比如说我嫌FF太闪眼了，我就可以sudo setpci -s 00:02.0 F4.B=CC，就会暗一些
