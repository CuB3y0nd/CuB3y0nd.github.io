---
title: 调用约定
subtitle: 更深入地了解 32-bit 和 64-bit 程序的参数
date: 2023-08-08T22:16:36+08:00
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

这篇文章将对 32-bit 和 64-bit 的调用约定进行更为深入的探讨

<!--more-->

{{< link href="/pwn_assets/calling-conventions-one-param.zip" content="calling-conventions-one-param.zip" title="Download calling-conventions-one-param.zip" download="calling-conventions-one-param.zip" card=true >}}

## 源码

让我们快速浏览一下源码：

```c {title="source.c"}
#include <stdio.h>

void vuln(int check) {
  if (check == 0xdeadbeef) {
    puts("Nice!");
  } else {
    puts("Not nice!");
  }
}

int main() {
  vuln(0xdeadbeef);
  vuln(0xdeadc0de);
}
```

分别运行 32-bit 和 64-bit 的 vuln，我们会得到相同的输出：

```
Nice!
Not nice!
```

## 分析 vuln-32

让我们用 radare2 对其进行反汇编：

```bash
$ r2 -d -A ./vuln-32
$ s main; pdf

0x080491ac      8d4c2404       lea ecx, [argv]
0x080491b0      83e4f0         and esp, 0xfffffff0
0x080491b3      ff71fc         push dword [ecx - 4]
0x080491b6      55             push ebp
0x080491b7      89e5           mov ebp, esp
0x080491b9      51             push ecx
0x080491ba      83ec04         sub esp, 4
0x080491bd      e832000000     call sym.__x86.get_pc_thunk.ax
0x080491c2      053e2e0000     add eax, 0x2e3e
0x080491c7      83ec0c         sub esp, 0xc
0x080491ca      68efbeadde     push 0xdeadbeef
0x080491cf      e88effffff     call sym.vuln
0x080491d4      83c410         add esp, 0x10
0x080491d7      83ec0c         sub esp, 0xc
0x080491da      68dec0adde     push 0xdeadc0de
0x080491df      e87effffff     call sym.vuln
0x080491e4      83c410         add esp, 0x10
0x080491e7      b800000000     mov eax, 0
0x080491ec      8b4dfc         mov ecx, dword [var_4h]
0x080491ef      c9             leave
0x080491f0      8d61fc         lea esp, [ecx - 4]
0x080491f3      c3             ret
```

如果我们仔细观察对 `sym.vuln` 的调用，我们会看到一个 Pattern：

```bash
push 0xdeadbeef
call sym.vuln
[...]
push 0xdeadc0de
call sym.vuln
```

在调用函数之前，我们实际上先将 `参数` 压入了栈。现在让我们来研究一下 `sym.vuln` 。

```bash
[0xf7fe3fd0]> db sym.vuln
[0xf7fe3fd0]> dc
INFO: hit breakpoint at: 0x8049162
[0x08049162]> pxw @ esp
0xffffd79c  0x080491d4 0xdeadbeef 0xf7c0c850 0xf7fc1380
```

第一个值是我之前的博客中提到的 `返回地址` ，而第二个值是 `参数` 。这是有道理的，
因为返回地址在 `调用` 期间被压入栈，因此它应该位于栈的顶部。现在让我们
反汇编 `sym.vuln` ：

```bash
┌ 74: sym.vuln (int32_t arg_8h);
│           ; arg int32_t arg_8h @ ebp+0x8
│           ; var int32_t var_4h @ ebp-0x4
│           0x08049162      55             push ebp
│           0x08049163      89e5           mov ebp, esp
│           0x08049165      53             push ebx
│           0x08049166      83ec04         sub esp, 4
│           0x08049169      e886000000     call sym.__x86.get_pc_thunk.ax
│           0x0804916e      05922e0000     add eax, 0x2e92
│           0x08049173      817d08efbead.  cmp dword [arg_8h], 0xdeadbeef
│       ┌─< 0x0804917a      7516           jne 0x8049192
│       │   0x0804917c      83ec0c         sub esp, 0xc
│       │   0x0804917f      8d9008e0ffff   lea edx, [eax - 0x1ff8]
│       │   0x08049185      52             push edx
│       │   0x08049186      89c3           mov ebx, eax
│       │   0x08049188      e8a3feffff     call sym.imp.puts           ; int puts(const char *s)
│       │   0x0804918d      83c410         add esp, 0x10
│      ┌──< 0x08049190      eb14           jmp 0x80491a6
│      │└─> 0x08049192      83ec0c         sub esp, 0xc
│      │    0x08049195      8d900ee0ffff   lea edx, [eax - 0x1ff2]
│      │    0x0804919b      52             push edx
│      │    0x0804919c      89c3           mov ebx, eax
│      │    0x0804919e      e88dfeffff     call sym.imp.puts           ; int puts(const char *s)
│      │    0x080491a3      83c410         add esp, 0x10
│      │    ; CODE XREF from sym.vuln @ 0x8049190(x)
│      └──> 0x080491a6      90             nop
│           0x080491a7      8b5dfc         mov ebx, dword [var_4h]
│           0x080491aa      c9             leave
└           0x080491ab      c3             ret
```

在这里，我显示了该命令的完整输出。因为其中有很多内容都是相关的。`radare2` 在
检测局部变量方面做得很好。正如你在顶部所看到的，有一个名为 `arg_8h` 。后来
又将 `arg_8h` 与 `0xdeadbeef` 进行比较：

```bash
cmp dword [arg_8h], 0xdeadbeef
```

显然这就是我们的参数。

现在我们知道，当有一个参数时，它会被压入栈，使栈看起来像：

```bash
return address        param_1
```

## 分析 vuln-64


