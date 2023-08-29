---
title: 连分数
subtitle:
date: 2023-08-29T12:31:05+08:00
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

连分数是一种将实数表示为一系列正整数的方法。

<!--more-->

## 0x01 概述

连分数是一种将实数表示为一系列正整数的方法。假设我们有一个实数 $\alpha_{1}$ 。
我们可以通过以下方式从 $\alpha_{1}$ 的带一系列正整数：

$$\alpha_{1} = \alpha_{1} + \frac{1}{\alpha_{2}} ,\ 其中\ \alpha_{1} = \lfloor{x}\rfloor$$

假设 $\alpha_{1} = 1.5$ ，我们可以说：

$$\alpha_{1} = 1 + \frac{1}{2} ,\ 其中\ \alpha_{2} = 2$$

这里的技巧在于：如果 $\alpha_{2} \notin \mathbb{Z}$ ，我们可以以 $\alpha_{2}$ 为基础，
继续这个精确的过程，从而保持连分数的进行。

$$\alpha_{1} = \alpha_{1} + \frac{1}{\alpha_{2} + \frac{1}{\alpha_{3} + \frac{1}{\alpha_{4}}}}$$

## 0x02 例子

下面是一个 $\frac{17}{11}$ 的例子：

$$\frac{17}{11} = 1 + \frac{6}{11} = 1 + \frac{1}{\frac{11}{6}} = 1 + \frac{1}{1 + \frac{5}{6}} = 1 + \frac{1}{1 + \frac{1}{\frac{6}{5}}} = 1 + \frac{1}{1 + \frac{1}{1 + \frac{1}{5}}}$$

在这个例子中，将连分数列表表示为系数列表是这样的：

$$\frac{17}{11} = [1; 1, 1, 5]$$

## 0x03 收敛子（渐进分数）

连分数的第 $k$ 个收敛子是通过截断连分数并仅使用序列的前 $k$ 项所获得的分数近似值。例如 $\frac{17}{11}$
的第 2 个收敛子是 $1 + \frac{1}{1} = \frac{2}{1} = 2$ ，而第 3 个收敛子是 $1 + \frac{1}{1 + \frac{1}{1}} = 1 + \frac{1}{2} = \frac{3}{2}$ 。

这些收敛子的一个应用是作为 **无理的无限连分数的有理数逼近**。

### 1x01 收敛子的性质

- 作为一个序列，它们有一个极限
  - 这个极限是 $\alpha_{1}$，你试图逼近的实数
- 它们交替大于和小于 $\alpha_{1}$

## 0x04 Sage

在 Sage 中，我们可以很容易的定义出连分数：

```python
f = continued_fraction(17/11)
```

然后我们可以打印系数列表：

```python
print(f)
>>> [1; 1, 1, 5]
```

甚至还为我们计算了收敛子：

```python
print(f.convergents())
>>> [1, 2, 3/2, 17/11]
```

