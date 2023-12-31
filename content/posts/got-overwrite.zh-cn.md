---
title: GOT Overwrite
subtitle: 劫持函数
date: 2023-08-28T19:54:34+08:00
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

GOT Overwrite 思路。

<!--more-->

GOT 表存储了动态链接函数的真实地址。如果能修改其中的某个函数地址，就可以控制程序
的执行流程。

看一个简单的例子：

```c
char buffer[20];
gets(buffer);
printf(buffer);
```

上面的示例代码中不仅存在缓冲区溢出漏洞，还存在格式化字符串漏洞。我们可以利用格式化字符串
漏洞将 `printf` 的 GOT 表项为覆写为 `system` 函数的地址。

这样覆写后的代码在执行时就相当于：

```c
char buffer[20];
gets(buffer);
system(buffer);
```

在覆写 `printf` 为 `system` 的情况下，原本调用 `printf` 打印用户输入的地方变成了直接将
用户的输入作为参数调用 `system` 。

虽然我们成功修改了 GOT 表，控制了程序的执行流，但用户的输入没有过滤就直接拿来调用 `system`
可能存在问题，需要注意传入的参数。

