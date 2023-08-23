---
title: 环、域和欧拉函数
subtitle: 环、域和欧拉函数 $\varphi$ 的基础知识
date: 2023-08-10T19:48:11+08:00
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
  - Crypto
  - Algebra
  - Mathematics
  - Number Theory
categories:
  - Cryptography
hiddenFromHomePage: false
hiddenFromSearch: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: true
lightgallery: false
password:
message:
repost:
  enable: false
  url:
---

环、域和欧拉函数 $\varphi$ 的基础知识

<!--more-->

## 0x01 环

在模运算中，我们对某个模 $m$ 进行取模，我们可以将能够以这种方式使用的数字视为
一组数字：

$$\mathbb{Z}/m\mathbb{Z} = \\{0, 1, 2, ..., m - 1\\}$$

该环 $\mathbb{Z}/m\mathbb{Z}$ 是 *模 $m$ 的整数环* 。我们可以对这个环的元素
进行加法和乘法操作，然后除以 $m$ 并取余数以获得 $\mathbb{Z}/m\mathbb{Z}$ 中
的元素。有一些 [通用规则](https://zh.wikipedia.org/wiki/%E7%8E%AF_(%E4%BB%A3%E6%95%B0)) 将其定义为环，包括关联加法和乘法的需要。

### 环的单位

正如我们所讨论的，如果 $gcd(a, m) = 1$，$a \in \mathbb{Z}/m\mathbb{Z}$ 具有模逆元。
所有具有模逆元的数字的集合表示为 $(\mathbb{Z}/m\mathbb{Z})^{*}$ ，这称为模 $m$ 的
单位群。有倒数的数称为 **单位**。例如：

$$(\mathbb{Z}/12\mathbb{Z})^{*} = \\{1, 5, 7, 11\\}$$

你可以将模 $12$ 的单位组视为本质上包含所有小于 $12$ 且与其互质的数字的集合。

## 0x02 域

如果 $\mathbb{Z}/m\mathbb{Z}$ 中的每个数都有模逆元，则它将从 **环** 提升为 **域**（域是一种
可以做除法的环）。请注意，唯一可能的 $m$ 值是素数，这就是这些字段通常表示为 $F_{p}$
的原因。**有限域（伽罗瓦域）** 是具有有限数量元素的域，例如模运算中使用的域。因此，
有限域这个术语会经常出现来描述 $F_{p}$ 。

### 相关集合

$\mathbb{Z}$ 表示 **整数环** 。请注意它不是一个域，因为没有分数可以提供逆元。

## 0x03 欧拉函数

欧拉函数返回单位组中以 $m$ 为模的元素数量。从数学上来说，这意味着：

$$\phi(m) = \\#(\mathbb{Z}/m\mathbb{Z})^{*} = \\#\\{0 \le a < m : gcd(a, m) = 1\\}$$

这个函数有大量的应用程序和进一步的规则，使我们能够做一些非常棒的事情，并且是 RSA 密码
系统的关键部分。

### 1x01 总计规则

如果 $gcd(p, q) = 1$ 则 $\phi(pq) = \phi(p)\phi(q)$ 。

### 1x02 计算欧拉函数

如果我们说 $n$ 的质因数分解是 $n = p_{1}^{e_{1}} \cdot p_{2}^{e_{2}} \cdot ... \cdot p_{k}^{e_{k}}$ 那么：

$$\phi(n) = n \prod_{p \mid n}(1 - \frac{1}{p})$$

请注意，如果 $p$ 是素数，则 $\phi(p) = p − 1$ 。

**证明**

TODO

### 1x03 欧拉公式

欧拉公式指出，如果 $gcd(a, p) = 1$ 那么：

$$a^{\phi(n)} \equiv 1 \mod n$$

这可能是 RSA 密码系统最重要的公式，我们很快就会讲到。请注意，这实际上告诉我们
更多关于以素数 $p$ 为模的情况，这种情况以费马命名。

### 1x04 费马小定理

由于 $\phi(p) = p − 1$，费马小定理指出：

$$a^{p-1} \equiv 1 \mod p$$

这是费马在欧拉公式提出前 100 多年提出的。你可能会注意到，这也为我们提供了一种
快速计算 $Fp$ 中 $a$ 的模反元素的方法：

{{<raw>}}

\[a^{p-1} \equiv 1 \mod p \\
a^{p-1} \cdot a^{-1} \equiv 1 \cdot a^{-1} \mod p \\
a^{p-2} \equiv a^{-1} \mod p\]

{{</raw>}}




