---
title: Gadgets
subtitle: 使用代码片段控制执行
date: 2023-08-11T10:56:47+08:00
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
lightgallery: true
password:
message:
repost:
  enable: false
  url:
---

Gadgets 是一小段后跟 `ret` 指令的代码。例如：`pop rdi; ret` 。我们可以操纵这些
gadgets 的 `ret` ，将它们串成一个链来完成我们想要做的事情。

<!--more-->

## 0x01 例子

假设在执行 `pop rdi; ret` gadget 期间栈看起来像这样：

<div align=center>
  <image src="https://s1.ax1x.com/2023/08/11/pPneJr8.png"/>
</div>

很明显：`0x10` 被弹出到 `rdi` 中，因为它在弹出到 `rdi` 前位于栈顶。一旦出栈，
`rsp` 就会移动：

<div align=center>
  <image src="https://s1.ax1x.com/2023/08/11/pPneGKf.png"/>
</div>

由于 `ret` 相当于 `pop rip` ，因此 `0x5655576724` 被移至 `rip` 中。
请注意栈是如何布局的。

## 0x02 使用 Gadgets

当我们覆盖返回地址时，我们覆盖了 `rsp` 指向的值。一旦该值出栈，它就会
指向栈中的下一个值。但请等待，我们可以覆盖栈中的下一个值。

假设我们想要利用二进制文件跳转到 `pop rdi; ret` gadget，将 `0x100` 弹出到
`rdi` 中，然后跳转到 `flag()` 。让我们一步步执行：

<div align=center>
  <image src="https://s1.ax1x.com/2023/08/11/pPneYqS.png"/>
</div>

在原来的 `ret` 上，我们覆盖了返回地址，我们让 gadget 地址出栈。现在 `rip`
移动指向到 gadget，`rsp` 移动到下一个内存地址。

<div align=center>
  <image src="https://s1.ax1x.com/2023/08/11/pPneNVg.png"/>
</div>

`rsp` 移动指向到 `0x100`；`rip` 变为 `pop rdi`。现在，当我们出栈时，`0x100`
被移入 `rdi` 。

<div align=center>
  <image src="https://s1.ax1x.com/2023/08/11/pPne3xP.png"/>
</div>

RSP 移动到栈上的下一个项目，即 `flag()` 的地址。执行 `ret` 并调用 `flag()` 。

## 0x03 总结

本质上，如果 gadget 从栈中弹出值，只需将这些值放在后面（包括 `ret` 中
的 `pop rip`）。如果我们想将 `0x10` 弹出到 `rdi` 然后跳转到 `0x16` ，
我们的 payload 将如下所示：

<div align=center>
  <image src="https://s1.ax1x.com/2023/08/11/pPneUaQ.png"/>
</div>

注：如果你有多个 `pop` 指令，则可以添加更多值。

<div align=center>
  <image src="https://s1.ax1x.com/2023/08/11/pPnea5j.png"/>
</div>

{{<admonition type="info">}}

之所以使用 `rdi` 作为示例，是因为如果你还记得的话，那是 64-bit 程序中
的第一个参数的寄存器。这意味着使用该 gadget 控制该寄存器非常重要。

{{</admonition>}}

## 0x04 查找 Gadgets

我们可以使用 [ROPGadget](https://github.com/JonathanSalwan/ROPgadget) 工具来查找可能的 gadgets 。

```
$ ROPgadget --binary vuln-64

Gadgets information
============================================================
0x0000000000401069 : add ah, dh ; nop dword ptr [rax + rax] ; ret
0x000000000040109b : add bh, bh ; loopne 0x401105 ; nop ; ret
0x0000000000401037 : add byte ptr [rax], al ; add byte ptr [rax], al ; jmp 0x401020
[...]
```

将其与 `grep` 结合起来查找特定的寄存器：

```
$ ROPgadget --binary vuln-64 | grep rdi

0x0000000000401096 : or dword ptr [rdi + 0x404030], edi ; jmp rax
0x00000000004011db : pop rdi ; ret
```

