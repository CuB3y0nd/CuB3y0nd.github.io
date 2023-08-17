---
title: 德布鲁因（De Bruijn）序列
subtitle: 计算偏移量的方法
date: 2023-08-08T02:30:16+08:00
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

`n 阶` [De Bruijn 序列](https://en.wikipedia.org/wiki/De_Bruijn_sequence) 是一个由 `n` 个不重复的字符组成的字符串序列。这使得查找 EIP
之前的偏移量变得更加简单：我们只需传入 De Bruijn 序列，获取 EIP 中的值并找到
序列中的 **一个可能的匹配** 来计算偏移量。这里将在 **ret2win** 二进制文件上执行此操作。

<!--more-->

{{< link href="/pwn_assets/ret2win.zip" content="ret2win.zip" title="Download ret2win.zip" download="ret2win.zip" card=true >}}

## 0x01 生成序列

同样，`pwndbg` 附带了一个很好的命令行工具（称为 `cyclic`），可以为我们生成它。
让我们创建一个长度为 `100` 的序列。

```
$ cyclic 100
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa
```

{{<admonition type="tip">}}

直接使用 cyclic 将生成默认长度为 100 的序列，指定序列长度可以使用 `cyclic <count>` 。

{{</admonition>}}

## 0x02 使用序列

现在我们有了序列，让我们在 `pwndbg` 提示输入时将其输入，使程序崩溃，然后计算
EIP 沿着序列有多远。

```
$ pwndbg vuln
$ r
Overflow me
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa
$ c
[...]
Program received signal SIGSEGV, Segmentation fault.
0x6161616e in ?? ()
[...]
```

崩溃的地址是 `0x6161616e` ；我们可以给 `cyclic` 加上参数 `-l <lookup_value>` 来计算偏移量：

```
$ cyclic -l 0x6161616e
Finding cyclic pattern of 4 bytes: b'naaa' (hex: 0x6e616161)
Found at offset 52
```

成功得到了正确的偏移地址！

