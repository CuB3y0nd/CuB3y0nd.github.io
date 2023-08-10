---
title: No eXecute
subtitle: 针对 shellcode 的防御
date: 2023-08-08T20:40:51+08:00
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
lightgallery: true
password:
message:
repost:
  enable: false
  url:
---

正如你所预料的那样，程序员对于人们可以将自己的指令注入到程序中 ~~*感到非常高兴（bushi*~~
NX 位代表「不可执行」，将内存区域定义为 **指令** 或 **数据**，把需要写入数据的内存
标识为可写，把保存指令的内存标识为可执行，但不会有一块内存同时被标识为可写
和可执行。这意味着你的输入将被存储为 **数据**，任何将其作为指令运行的尝试
都会使程序崩溃，从而有效地阻止 shellcode 。

为了绕过 NX，我们必须使用一种被称为 **ROP** ，Return-Oriented Programming（面向返回
编程）的技术。

<!--more-->

{{<admonition type="info">}}

Windows 版本的 NX 是 DEP（**D**ata **E**xecution **P**revention），数据执行保护。

{{</admonition>}}

## 检查 NX

你可以使用 `checksec` 或 `rabin2` 来检查 NX 的状态：

```
$ checksec --file=vuln
RELRO         STACK CANARY    NX          PIE    RPATH    RUNPATH    Symbols    FORTIFY Fortified Fortifiable FILE
Partial RELRO No canary found NX disabled No PIE No RPATH No RUNPATH 67 Symbols No      0         1           vuln
```

</br>

```
$ rabin2 -I vuln
[...]
nx       false
[...]
```
