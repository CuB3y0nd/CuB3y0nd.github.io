---
title: 面向返回编程简介
subtitle: 绕过 NX
date: 2023-08-08T22:03:44+08:00
draft: false
author:
  name: CuB3y0nd
  link:
  email:
  avatar:
description:
keywords:
license:
comment: true
weight: 0
tags:
  - PWN
  - Stack
categories:
  - PWN
hiddenFromHomePage: false
hiddenFromSearch: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: false
math: false
lightgallery: false
password:
message:
repost:
  enable: false
  url:
---

ROP 的基础是将二进制文件文件本身中已经存在的代码块链接在一起，以便执行你想要
的操作。这通常涉及将参数传递给 `libc` 中已经存在的函数，例如 `system()` 。如果你
可以找到命令的位置，例如 `cat flag.txt`，然后将其 *作为参数* 传递给 `system()` 函数，
它将执行该命令并返回输出。一个更危险的命令是 `/bin/sh`，当 `system()` 运行它时，
它会为攻击者提供一个 shell，就像我们使用的 shellcode 一样。

然而，这样做并没有看起来那么简单。为了能够正确调用函数，我们首先必须了解如何
向函数传递参数。

<!--more-->
