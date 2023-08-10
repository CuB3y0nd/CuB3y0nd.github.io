---
title: 调用约定
subtitle: 更深入地了解 32-bit 和 64-bit 程序的参数
date: 2023-08-09T10:16:36+08:00
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

## 单个参数

{{< link href="/pwn_assets/calling-conventions-one-param.zip" content="calling-conventions-one-param.zip" title="Download calling-conventions-one-param.zip" download="calling-conventions-one-param.zip" card=true >}}

### 0x01 源码

我们先快速浏览一下源码：

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

### 0x02 分析 vuln-32

用 radare2 对其进行反汇编：

```
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

```
push 0xdeadbeef
call sym.vuln
[...]
push 0xdeadc0de
call sym.vuln
```

在调用函数之前，我们实际上先将 `参数` 压入了栈。现在让我们来研究一下 `sym.vuln` 。

```
[0xf7fe3fd0]> db sym.vuln
[0xf7fe3fd0]> dc
INFO: hit breakpoint at: 0x8049162
[0x08049162]> pxw @ esp
0xffffd79c  0x080491d4 0xdeadbeef 0xf7c0c850 0xf7fc1380
```

第一个值是我之前的博客中提到的 `返回地址` ，而第二个值是 `参数` 。这是有道理的，
因为返回地址在 `调用` 期间被压入栈，因此它应该位于栈顶。现在让我们反汇编 `sym.vuln` ：

```
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
检测局部变量方面做得很好。正如你在顶部所看到的，有一个名为 `arg_8h` 的局部变量。
后来，又将 `arg_8h` 与 `0xdeadbeef` 进行比较：

```
cmp dword [arg_8h], 0xdeadbeef
```

因此可以分析出 `arg_8h` 就是我们的参数。

现在我们知道，当有一个参数时，它会被压入栈，使栈看起来像：

```
return address        param_1
```

### 0x03 分析 vuln-64

我们在这里再次反汇编 `main` ：

```
0x00401153      55             push rbp
0x00401154      4889e5         mov rbp, rsp
0x00401157      bfefbeadde     mov edi, 0xdeadbeef
0x0040115c      e8c1ffffff     call sym.vuln
0x00401161      bfdec0adde     mov edi, 0xdeadc0de
0x00401166      e8b7ffffff     call sym.vuln
0x0040116b      b800000000     mov eax, 0
0x00401170      5d             pop rbp
0x00401171      c3             ret
```

我们发现 64-bit 和 32-bit 在传参上不一样了。正如我在这篇 [博客](https://www.cubeyond.net/32-bit-vs-64-bit/) 中所说的，
参数被移至 `rdi`（这里的反汇编中写的是 `edi` ，但 `edi` 只是 `rdi` 的低 32 bits
寄存器。原因是我们传入的参数只有 32 bits 大小，所以改为 `EDI` 可以节省
内存消耗）。如果我们再次中断 `sym.vuln` ，我们可以使用以下命令检查 `rdi` 。

```
$ dr rdi
```

{{<admonition type="info">}}

如果只使用 `dr` 则会显示所有寄存器。

{{</admonition>}}

```
[0x00401153]> db sym.vuln
[0x00401153]> dc
INFO: hit breakpoint at: 0x401122
[0x00401122]> dr rdi
0xdeadbeef
```

{{<admonition type="info">}}

64-bit 程序中，寄存器用于存放参数。但返回地址仍然压入栈，并且在 ROP 中放置在
函数地址之后。

*注：只有前六个参数才会分别保存在寄存器中，如果还有更多的参数的话则会保存在栈上。*

{{</admonition>}}

## 多个参数

{{< link href="/pwn_assets/calling-convention-multi-param.zip" content="calling-convention-multi-param.zip" title="Download calling-convention-multi-param.zip" download="calling-convention-multi-param.zip" card=true >}}

### 0x01 源码

```c {title="source.c"}
#include <stdio.h>

void vuln(int check, int check2, int check3) {
  if (check == 0xdeadbeef && check2 == 0xdeadc0de && check3 == 0xc0ded00d) {
    puts("Nice!");
  } else {
    puts("Not nice!");
  }
}

int main() {
  vuln(0xdeadbeef, 0xdeadc0de, 0xc0ded00d);
  vuln(0xdeadc0de, 0x12345678, 0xabcdef10);
}
```

### 0x02 分析 vuln-32

由于我们之前已经看到了几乎相同的二进制文件的完整反汇编，因此在这里我只会列出
重要的部分：

```
0x080491dd      680dd0dec0     push 0xc0ded00d
0x080491e2      68dec0adde     push 0xdeadc0de
0x080491e7      68efbeadde     push 0xdeadbeef
0x080491ec      e871ffffff     call sym.vuln
[...]
0x080491f7      6810efcdab     push 0xabcdef10
0x080491fc      6878563412     push 0x12345678
0x08049201      68dec0adde     push 0xdeadc0de
0x08049206      e857ffffff     call sym.vuln
```

我们发现 `压栈` 和 `传参` 顺序是相反的。这是因为取参的时候是从低地址向高地址取参，
而先入栈的在高地址，正好符合了取参从低向高的规则。

{{<admonition type="info">}}

大多数计算机系统结构中，栈是一种后进先出（Last In First Out，LIFO）的
数据结构。当程序调用一个函数时，函数的参数被压入栈中，而函数内部则可以
按照相反的顺序逐个弹出这些参数进行处理。这种设计有几个原因：

1. 便于管理栈指针：压栈和出栈操作可以通过简单的栈指针操作来实现，无需
复杂的数据重排。这样可以减小指令的数量和复杂度，提高执行效率。
2. 一致性：使用相同的栈结构来处理参数和局部变量可以简化函数调用和返回
的实现，使得代码更加一致和可维护。
3. 节省存储空间：压栈和出栈操作可以在相对较小的内存区域进行，不需要预留
很大的内存来存储参数，这有助于节省内存空间。

举个例子：如果一个函数有三个参数：`a`、`b` 和 `c` ，调用函数时的顺序为 `func(c, b, a)` ，
则在栈中的存储顺序为 `push a` ，`push b` ，`push c` ，而在函数内部获取
参数的顺序为从栈顶依次弹出 `pop c`，`pop b`，`pop a`。

虽然压栈和实际程序传参的顺序相反，但这种细节是由编译器和计算机体系结构
来处理的，因此我们无需过多考虑。编译器会生成适当的指令来正确处理函数
参数的压栈和出栈操作，以确保函数调用的正确执行。

{{</admonition>}}

```
[0x080491bf]> db sym.vuln
[0x080491bf]> dc
INFO: hit breakpoint at: 0x8049162
[0x08049162]> pxw @ esp
0xff80358c  0x080491f1 0xdeadbeef 0xdeadc0de 0xc0ded00d
```

因此，如何将更多参数放置在栈上就变得非常清楚了：

```
return address        param1        param2        param3        [...]        paramN
```

### 0x03 分析 vuln-64

```
0x00401170      ba0dd0dec0     mov edx, 0xc0ded00d
0x00401175      bedec0adde     mov esi, 0xdeadc0de
0x0040117a      bfefbeadde     mov edi, 0xdeadbeef
0x0040117f      e89effffff     call sym.vuln
0x00401184      ba10efcdab     mov edx, 0xabcdef10
0x00401189      be78563412     mov esi, 0x12345678
0x0040118e      bfdec0adde     mov edi, 0xdeadc0de
0x00401193      e88affffff     call sym.vuln
```

同理，根据上面的调试步骤查看寄存器内容，我们可以发现：除了 `rdi` 之外，
我们还把参数压到了 `rsi` 和 `rdx` 。

## 更大的 64-bits 值

只是为了表明实际上最终使用的是 `rdi` 而不是 `edi` ，我将更改原始的单参数
代码以使用更大的数字：

```c {title="source.c"}
#include <stdio.h>

void vuln(long check) {
  if (check == 0xdeadbeefc0dedd00d) {
    puts("Nice!");
  }
}

int main() {
  vuln(0xdeadbeefc0dedd00d);
}
```

如果你反汇编 `main` ，你可以看到它被反汇编为：

```
movabs rax, 0xdeadbeefc0ded00d
mov rdi, rax
call sym.vuln
```

{{<admonition type="info">}}

`movabs` 用于将 `mov` 指令编码为 64-bit 指令，可将其视为 `mov` 。

{{</admonition>}}

