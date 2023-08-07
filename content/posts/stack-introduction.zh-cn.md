---
title: 二进制漏洞利用简介
subtitle:
date: 2023-08-07T17:26:49+08:00
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

**二进制利用** 是指发现程序中的漏洞并利用它们来执行你想要的操作。有时，这可能会导致
绕过身份验证或机密信息泄露，但偶尔（如果幸运的话）它也可能导致远程代码执行（RCE）。
二进制利用的最基本形式发生在 **栈** 上，栈是存储代码中函数创建的临时变量的内存区域。

当调用一个新函数时，被调用函数的内存地址被压入栈——这样，程序就知道被调用函数
执行完成后应该返回到哪里。让我们看一个基本的二进制文件来展示这一点。

<!--more-->

{{< link href="/pwn_assets/introduction.zip" content="introduction.zip" title="Download introduction.zip" download="introduction.zip" card=true >}}

## 0x01 分析

该压缩包有两个文件 —— `source.c` 和 `vuln`；后者是 `ELF` 文件，是 Linux 的
可执行文件格式。

这里将使用 `radare2` 来分析调用函数时二进制文件的行为。

```bash
$ r2 -d -A ./vuln
```

`-d` 调试可执行文件</br>
`-A` 执行分析

我们可以反汇编 `main` 函数：

```bash
$ s main; pdf
```

`s main` 用于定位（跳转）到 `main` 函数，而 `pdf` 表示 反汇编（**P**rint **D**isassemble **F**unction）。

```
0x080491ab      55             push ebp
0x080491ac      89e5           mov ebp, esp
0x080491ae      83e4f0         and esp, 0xfffffff0
0x080491b1      e80d000000     call sym.__x86.get_pc_thunk.ax
0x080491b6      054a2e0000     add eax, 0x2e4a
0x080491bb      e8b2ffffff     call sym.unsafe
0x080491c0      90             nop
0x080491c1      c9             leave
0x080491c2      c3             ret
```

对 `unsafe` 的调用位于 `0x080491bb`，我们可以在那里设一个断点。

```bash
$ db 0x080491bb
```

`db` 表示 **d**ebug **b**reakpoint，设置断点。断点的作用是在到达时暂停程序的执行以便
运行其它命令。现在我们运行 `dc`，**d**ebug **c**ontinue，继续运行程序，直到遇到断点
时暂停。

它应该在调用 `unsafe` 之前暂停；现在我们来分析一下栈顶：

```bash
[0x080491ab]> pxw @ esp
0xffcac570  0x00000000        [...]
```

{{<admonition type="info">}}

`ESP 寄存器` 是栈指针寄存器，其内存放着一个指针，该指针永远指向系统栈最上面一个
栈帧的栈顶。

{{</admonition>}}

第一个地址 `0xffcac570` 是位置；`0x00000000` 是该位置的值。让我们用 `ds`，**d**ebug **s**tep，
步入指令，并再次检查栈顶。

```bash
[0x08049172]> pxw @ esp
0xffcac56c  0x080491c0 0x00000000
```

可以发现值 `0x080491c0` 被压入栈顶，它应该出现在二进制文件中：

```
[...]
0x080491b6      054a2e0000     add eax, 0x2e4a
0x080491bb      e8b2ffffff     call sym.unsafe
0x080491c0      90             nop
[...]
```

不难发现，这里其实是调用了 `unsafe` 之后的指令。这说明了程序是如何知道 `unsafe()`
执行完成后应该返回到哪里的。

## 0x02 漏洞

现在让我们看看如何破解这个程序。首先，我们反汇编 `unsafe` 并在 `ret` 指令处进行中断；
`ret` 相当于 `pop eip`，它将把我们刚才分析的栈上保存的返回指针 `0x080491c0` 压入
`eip 寄存器` 中。

{{<admonition type="info">}}

`EIP 寄存器` 用来存储 CPU 要读取的指令的地址，CPU 通过 `EIP 寄存器` 读取即将要
执行的指令。

每次 CPU 执行完相应的汇编指令后，`EIP 寄存器` 的值就会增加。

{{</admonition>}}

现在让我们继续将一堆字符发送到输入中，看看这会对它有什么影响：

```bash
[0x08049172]> db 0x080491aa
[0x08049172]> dc
Overflow me
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

现在让我们读取返回指针之前所在位置的值，我们看到它是 `0xffcac56c` 。

```bash
[0x080491aa]> pxw @ 0xffcac56c
0xffcac56c  0x41414141 0x41414141 0x41414141 0x41414141  AAAAAAAAAAAAAAAA
```

我们发现栈上内容全部变成了 `0x41414141`，覆盖了原先的返回指针 `0x080491c0`

原理很简单：由于我们输入的数据比程序预期的要多，这导致我们覆盖的栈也比程序
预期的要多。因为保存的返回指针也在栈上，这意味着我们设法覆盖它。结果，`ret`
指令本来应该压入 `eip` 中的值被改变，不会在前面的函数中执行，而是执行了
`0x41414141` 。

我们可以用 `ds` 确认。

```bash
[0x080491aa]> ds
[0x41414141]>
```

下一条将要执行的指令的位置变成了：`0x41414141`。让我们运行 `dr eip` 以确保
`0x41414141` 是 `eip` 中的值：

```bash
[0x41414141]> dr eip
0x41414141
```

我们已经成功劫持了程序的执行流程！让我们试试当我们用 `dc` 继续运行时，
它是否会崩溃。

```bash
[0x41414141]> dc
[+] SIGNAL 11 errno=0 addr=0x41414141 code=1 si_pid=1094795585 ret=0
```

正如我们所料，程序果然崩溃了。这说明我们成功找到了这个程序的漏洞，并且利用
漏洞破坏了程序的执行流程。

`radare2` 非常有用，它会打印出导致程序崩溃的地址。如果程序在调试器之外崩溃，
它通常会显示「分段错误」。这可能意味着多种情况，但通常是你已经覆盖了 `EIP` 。

{{<admonition type="info" title="修复">}}

当然，你可以防止人们在使用程序时输入比预期更多的字符，通常使用其他 C 函数
即可解决问题。例如 `fgets()`；`gets()` 本质上是不安全的，因为它不检查输入的长度。
你始终应该确保程序中没有使用诸如 `gets()` 这样危险的函数。你也可能给 `fgets()`
提供错误的参数，导致它仍然接受太多字符。

{{</admonition>}}

## 0x03 总结

当一个函数调用另一个函数时：

- 将返回指针压入栈，以便被调用的函数知道返回到哪里
- 当被调用函数执行完成时，它再次将其从栈中弹出

因为这个值保存在栈上，就像我们的局部变量一样，如果我们写入的字符比程序预期的多，
我们可以覆盖该值并将代码执行重定向到我们希望的任何地方。`fgets()` 等函数可以防止
这种简单的溢出，但你始终应该检查程序实际读取了多少内容。
