---
title: 更可靠的 shellcode
subtitle: 无需猜测工作的 shellcode
date: 2023-08-28T21:10:57+08:00
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

通过寄存器使用 shellcode

<!--more-->

## 0x01 利用 ROP

目前 shellcode 漏洞利用中存在的问题是 shellcode 的位置不确定。如果我们可以控制
写入 shellcode 的位置，那将会使编写 exp 变得非常方便。

其实我们可以这样做：

与直接写入 shellcode 不同，我们可以利用 ROP 再次输入，指定 shellcode 的写入位置
为我们想要控制的寄存器。

## 0x02 使用 ESP

请你想一想，返回地址从栈中弹出后，ESP 会指向其在内存中的下一个位置。如果我们把
shellcode 放在返回地址后面，那么这个 shellcode 将会被执行，从而实现注入。

我们可以用 `jmp esp` gadget 来控制返回地址，ESP 会指向返回地址之后的 shellcode。
返回地址出栈后，ESP 会指向 shellcode，然后 `jmp esp` 会直接跳转到 shellcode 并执行它。

## 0x03 ret2reg

ret2reg 将 `jmp esp` 的利用推广到可以使用任意寄存器进行跳转。

也就是说，不仅可以使用 `jmp esp` 来跳转到 ESP 寄存器所指向的 shellcode 位置，
还可以利用 `jmp` 其它寄存器的 gadget 来跳转到 shellcode 。

只要我们控制某个寄存器指向 shellcode，就可以构造出一个 ret2reg 的利用链。

