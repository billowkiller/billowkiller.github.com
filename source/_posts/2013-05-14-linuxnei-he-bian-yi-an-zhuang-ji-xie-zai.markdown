---
layout: post
title: "linux内核编译安装及卸载"
date: 2013-05-14 01:09
comments: true
categories: linux
tags: [linux,kernel,compile]
---

编译安装：

下载需要的包

{% codeblock lang:bash %}

apt-get install kernel-package libncurses5-dev fakeroot wget bzip2
cp linux-3.x.x /usr/src
cd /usr/src/linux-3.x.x
make menuconfig
make modules
make modules\_install
make
make install
sudo mkinitramfs -o /boot/initrd.img-3.x.x
sudo update-initramfs -c -k 3.x.x
sudo update-grub2
{% endcodeblock %}

* * * * *

 <!--more-->

卸载：

 custom compiled kernel you need to remove following files/dirs:

-   /boot/vmlinuz\*KERNEL-VERSION\*
-   /boot/initrd\*KERNEL-VERSION\*
-   /boot/System-map\*KERNEL-VERSION\*
-   /boot/config-\*KERNEL-VERSION\*
-   /lib/modules/\*KERNEL-VERSION\*/
-   Update grub configuration file /etc/grub.conf or /boot/grub/menu.lst
    to point to correct kernel version.
