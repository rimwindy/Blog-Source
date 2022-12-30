---
title: scp下载服务器文件
tags: [scp, 服务器]
date: 2019-11-2 18:46
categories: 笔记本
thumbnail: /img/2019/11/02/0.png
updated: 2022-11-20
---

{% note info Note %}

更推荐使用 rsync 替代 scp。

{% endnote %}

一个平常的晚上，突然心血来潮想试试 Blender 2.80 的 eevee，遂去官网看了下，没想到正式版都出来了，但下载安装包的时候却出现了极其蛋疼的问题：

![下载速度感人](/img/2019/11/02/1.png)

这显然是难以忍受的，正好我有一台洛杉矶的服务器，于是便打算利用它来下载，然后传回本地电脑上。相关的工具也很多，比如 xftp、scp、ftp 等，今天就记录一下 scp 的使用方法。

# 什么是 scp
scp 是 secure copy 的缩写, scp 是 linux 系统下基于 ssh 登陆进行安全的远程文件拷贝命令。和它类似的命令有 cp，不过 cp 只是在本机进行拷贝不能跨服务器，而且 scp 传输是加密的，可能会稍微影响一下速度。

# 常用命令

> 若ssh端口不是默认的22，可以加上-P port参数（注意-P为大写）

## 上传本地文件到服务器
```bash
scp /path/filename username@servername:/path/
```

## 从服务器上下载文件
```bash
scp username@servername:/path/filename /local_dir
```

## 从服务器下载整个目录
```bash
scp -r username@servername:/remote_dir /local_dir
```

## 上传目录到服务器
```bash
scp -r local_dir username@servername:remote_dir
```

# E.N.D
小忍姐姐真是太帅了~

*さようなら*