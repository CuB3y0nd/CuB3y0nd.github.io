---
title: 德布鲁因（De Bruijn）序列
subtitle: 更好的计算偏移量的方法
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
toc: true
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

同样，`radare2` 附带了一个很好的命令行工具（称为 ragg2），可以为我们生成它。
让我们创建一个长度为 `100` 的序列。

```bash
$ ragg2 -r -P 100
AAABAACAADAAEAAFAAGAAHAAIAAJAAKAALAAMAANAAOAAPAAQAARAASAATAAUAAVAAWAAXAAYAAZAAaAAbAAcAAdAAeAAfAAgAAh
```

`-r` 告诉它显示 ASCII 字节而不是十六进制对</br>
`-P` 指定长度

## 0x02 使用序列

现在我们有了序列，让我们在 `radare2` 提示输入时将其输入，使程序崩溃，然后计算
EIP 沿着序列有多远。

```bash
$ r2 -d -A ./vuln
[0xf7f19fd0]> dc
Overflow me
AAABAACAADAAEAAFAAGAAHAAIAAJAAKAALAAMAANAAOAAPAAQAARAASAATAAUAAVAAWAAXAAYAAZAAaAAbAAcAAdAAeAAfAAgAAh
[+] SIGNAL 11 errno=0 addr=0x41534141 code=1 si_pid=1095975233 ret=0
```

崩溃的地址是 `0x41534141` ；我们可以使用 `radare2` 的内置命令 `wopO` 来计算偏移量。

```bash
[0x41534141]> wopO 0x41534141
52
```

成功得到了正确的偏移量！

我们也可以偷懒，不复制崩溃地址：

```bash
[0x41534141]> wopO `dr eip`
52
```

反引号表示首先计算 `dr eip`，然后再对其结果运行 `wopO` 。
