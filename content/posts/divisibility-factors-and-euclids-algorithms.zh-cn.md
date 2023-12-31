---
title: 可除性，因数和欧几里得算法
subtitle: 数论基础概述
date: 2023-08-10T10:47:09+08:00
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

密码学建立在代数和数论的基础上，我希望在这里能充分介绍它们。

<!--more-->

## 0x01 可除性

如果存在一个整数 $c$ 使得 $a = bc$ ，则称 $a$ 可被 $b$ 整除。在这种情况下，
$b$ 被认为是 $a$ 的 **因数**。如果 $a$ 可以被 $b$ 整除，则可以用 $a \mid b$ 表示。
相反的，$a \nmid b$ 表示 $a$ 不可被 $b$ 整除。

## 0x02 最大公约数

给定两个整数 $a$ 和 $b$，最大公约数 $gcd(a, b)$（也称为最大公因数）是最大
整数 $p$ ，其中 $p \mid a$ 且 $p \mid b$ 。

### 1x01 欧几里得算法

给定 $a$ 和 $b$ ，我们可以写出连接两者的方程：

$$a = b \cdot q + r$$

其中 $q$ 和 $r$ 是符合 $r < b$ 方程的整数。基本上，$b$ 最多可分为 $q$ 次，
余数为 $r$ 。这就是窍门所在。

该方程中加在一起的每一项都必须能被 $gcd(a, b)$ 整除，因为如果我们将 $gcd$
视为 $g$ ，我们可以说 $a = k_{1}g, b = k_{2}g$：

$$k_{1}g = k_{2}g \cdot q + r$$

意味着 $r$ 必须是 $g$ 的整数倍。

但现在，如果我们跳出框框思考，我们就会意识到 $b$ 和 $r$ 都可以被 $gcd(a, b)$
整除。所以我们可以计算 $gcd(b, r)$ 。

这是一个很大的飞跃，需要一些思考，但让我们用代数的方式来分解它：

{{<raw>}}

\[ a = b \cdot q_{0} + r_{1} \\
b = r_{1} \cdot q_{1} + r_{2} \\
r_{1} = r_{2} \cdot q_{2} + r_{3} \\
r_{2} = r_{3} \cdot q_{3} + r_{4} \]

{{</raw>}}

等等，什么时候才能停止 GCD 呢？好吧，一旦 $r_{n} = 0$，我们就可以停止继续 GCD 了。
在这种情况下 $r_{n-2} \mid r_{n-1}$，因此我们可以将 $r_{n-1}$ 作为 GCD 。

{{<admonition type="info">}}

我强烈建议你仔细考虑一下，直到它对你有意义为止。

{{</admonition>}}

**例子**

假设我们要求 8075 和 16283 的 GCD 。首先，我们可以将其写成 $a = b \cdot q + r$ 的形式：

$$16283 = 8075 \cdot 2 + 133$$

现在我们尝试计算 8075 和 133 的 GCD 。

{{<raw>}}

\[ 8075 = 133 \cdot 60 + 95 \\
133 = 95 \cdot 1 + 38 \\
95 = 38 \cdot 2 + 19 \\
38 = 19 \cdot 2 + 0 \]

{{</raw>}}

因此，16283 和 8075 的 GCD 是 19 。

### 1x02 扩展欧几里得算法

我们可以进一步拓展欧几里得算法，除了 GCD 之外，还可以计算 $a, b$ 的 $u, v \in \mathbb{Z}$，
其总和等于 GCD，即：

$$au + bv = gcd(a, b)$$

这种拓展算法对于计算 **模反元素** 是非常有价值的。基于欧几里得算法计算 GCD，
然后将其写成其它数字，对最小的非 GCD 数字重复该过程，直到我们得出以下方程：
只有 GCD 和两个起始数字。让我们使用上面的例子，写出 GCD 的方程：

$$19 = 95 - 38 \cdot 2$$

现在 38 是最小的非 GCD 数，我们可以根据示例中的方程序列中的较大数来写它，然后
重复下一个最小的数：

{{<raw>}}

\[ 19 = 95 - 38 \cdot 2 \\
= 95 - (133 - 95 \cdot 1) \cdot 2 \\
= 95 - (133 \cdot 2 + 95 \cdot -2) \\
= 95 \cdot 3 + 133 \cdot -2 \\
= (8075 + 133 \cdot -60) \cdot 3 + 133 \cdot -2 \\
= 8075 \cdot 3 + 133 \cdot -180 + 133 \cdot -2 \\
= 8075 \cdot 3 + 133 \cdot -182 \\
= 8075 \cdot 3 + (16283 + 8075 \cdot -2) \cdot -182 \\
= 8075 \cdot 3 + 16283 \cdot -182 + 8075 \cdot 364 \\
= 8075 \cdot 367 + 16283 \cdot -182 \]

{{</raw>}}

因此我们的方程 $au + bv = gcd(a, b)$ 如下：

$$16283 \cdot -182 + 8075 \cdot 367 = 19$$

