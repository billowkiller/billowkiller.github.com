---
layout: post
title: "几个linux命令"
date: 2013-05-14 01:09
comments: true
categories: [linux, command]
---

转换文件到pdf unoconv -f pdf mydocument.odt\
zip乱码解压 unzip -O CP936 xx.zip\
mp3 乱码 mid3iconv -e gbk \*.mp3 //当前目录下的mp3文件\
视频转码 mencoder \*.rmvb -o output.avi -oac mp3lame -lameopts cbr:br=32
-ovc x264 -x264encopts bitrate=440 -vf scale=320:-3

linux 声音调节  alsamixer

gedit 乱码：

    gsettings set org.gnome.gedit.preferences.encodings auto-detected "['UTF-8','CURRENT','GB18030','ISO-8859-15','UTF-16']"

SSH 远程登录：   ssh -l root 192.168.0.150

vim 中文乱码：

let &termencoding=&encoding\
set fileencodings=utf-8,gbk
