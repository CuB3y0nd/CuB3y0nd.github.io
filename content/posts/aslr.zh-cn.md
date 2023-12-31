---
title: ASLR
subtitle:
date: 2023-08-23T09:53:49+08:00
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

地址空间布局随机化

<!--more-->

## 0x01 概述

ASLR（**A**ddress **S**pace **L**ayout **R**andomisation）代表地址空间布局随机化，在大多数
情况下，可以将其视为 libc 版本的 PIE 等价物。每次运行二进制文件时，libc（和其他库）
都会加载到不同的内存地址中。

{{<admonition type="warning" title="注意">}}

虽然人们很容易将 ASLR 视为 libc 版本的 PIE，但有一个关键的区别。

ASLR 是 **内核级保护**，而 PIE 是 **二进制文件级保护**。主要区别在于 **PIE 需要编译进二进制文件，
它存在于程序本身，运行环境无法修改**。而 ASLR 的存在完全 **取决于运行该二进制文件的环境，
可以被系统关闭**。如果程序在编译时关闭 ASLR，二进制文件就不包含 ASLR 随机化。即使在开启
ASLR 的环境中运行也无法对其产生效果。*即，ASLR 的作用需要在编译时就引入程序本身才行，
运行环境开启与否并不影响已经编译好的可执行文件。*

但是如果编译时开启了 ASLR，运行时关闭了 ASLR，会出现这样的情况：

如果程序被编译时已经引入了 ASLR 机制，那么程序本身就会包含实现随机化的元数据信息。
在运行环境中，系统级 ASLR 被关闭，就不会再引入额外的随机化。但是程序本身仍然会根据
自身签入的 ASLR 信息进行一次随机化。最终程序仍会有一定程度的地址随机化效果，但由于
缺乏系统级 ASLR 的叠加，随机范围会变小。

*但只有关闭 ASLR 后的第一次有一个小随机，第二次往后都是不变的。*

{{</admonition>}}

当然，与 PIE 一样，这意味着你无法对函数地址等值进行硬编码（例如 ret2libc 的 `system`）。

## 0x02 格式字符串陷阱

我们很容易想到：与 PIE 一样，我们可以简单地格式化 libc 地址的字符串并从中减去静态偏移量。
遗憾的是，我们还不能完全做到这一点。

当函数执行结束后，它们不会从内存中被删除，而是被忽略并重写。通过格式化字符串可能会获取到
这些遗留内容之一，但它们很可能已经失效，并不可靠。因此你获取的值可能根本不存在。即使存在，
偏移量也很可能不同（不同的 libc 具有不同的大小，导致函数之间的偏移量也不同）。你可能会幸运
的获取到正确的值，但你不应该期望偏移量保持一致。

更可靠的方法是读取 [特定函数的 GOT 条目](https://www.cubeyond.net/plt-and-got/#1x02-s-格式化字符串) 。

## 0x03 双重检验

出于与 PIE 相同的原因，libc 的基地址也始终以十六进制字符 `000` 结尾。

