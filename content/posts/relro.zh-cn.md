---
title: RELRO
subtitle: Relocation Read-Only
date: 2023-08-28T20:46:32+08:00
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

RELRO（Relocation Read-Only，重定位表只读）也是一种保护机制，可以有效防止 GOT 表被覆写。

<!--more-->

RELRO 有两种类型，原理都很简单。

#### 0x01 Partial RELRO

Partial RELRO 只是简单的把 GOT 表放到了程序变量的上面，通过调整 GOT 表在内存中
的位置来防止低地址向高地址的溢出攻击，这样可以防止通过缓冲区溢出等方式覆写 GOT 表。

但是这种保护并不能防止利用格式化字符串漏洞直接覆写 GOT 表的攻击。

#### 0x02 Full RELRO

Full RELRO 将整个 GOT 表设置为只读。这样的话，即使是格式化字符串漏洞也无法直接修改它。
由于它需要一次性解析所有函数地址，这可能会大大增加程序的加载时间，所以它不是一般可执行
文件的默认设置。

