---
title: 32-bit vs 64-bit
subtitle: 位数之间的差异
date: 2023-08-08T20:22:13+08:00
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
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: false
  url:
---

到目前为止我们所做的一切都适用于 64-bit 和 32-bit；你唯一需要更改的是将 `p32()`
改为为 `p64()` ，因为内存地址更长。

然而，两者之间的真正区别在于将参数传递给函数的方式（我们很快就会更仔细地讨论）；
在 32-bit 中，所有参数在调用函数之前都被压入栈。然而，在 64-bit 中，根据调用约定，
前 6 个参数分别存储在寄存器 `RDI`、`RSI`、`RDX`、`RCX`、`R8` 和 `R9` 中，如果超过
6 个，还有更多的参数的话则会保存在栈上。

<!--more-->

{{<admonition type="warning">}}

不同的操作系统有不同的 [调用约定](https://zh.wikipedia.org/wiki/X86%E8%B0%83%E7%94%A8%E7%BA%A6%E5%AE%9A) 。

{{</admonition>}}
