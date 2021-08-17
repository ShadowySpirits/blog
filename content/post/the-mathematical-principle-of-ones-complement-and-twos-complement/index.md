---
title: 反码和补码的数学原理
slug: the-mathematical-principle-of-ones-complement-and-twos-complement
date: 2021-08-16T19:18:16+08:00
tags:
  - Math
categories:
  - 某不存在的技术
---

本文介绍了使用反码和补码的加法代替减法，并分析了这样做背后的数学原理

<!--more-->

{{< tip warning >}}
本文公式较多，建议使用电脑或平板阅读
{{< /tip >}}

## 反码与补码的表示

1. 原码的表示方法：
   符号位加上它的绝对值，即用第一位表示符号，其余位表示值。如果是 $8$ 位二进制：
   <div>
   $$
   \begin{array}{l}
   [+1] = [00000001]_原 \\
   [-1] = [10000001]_原
   \end{array}
   $$
   </div>

2. 反码的表示方法：
   正数的反码是其本身
   负数的反码是在其原码的基础上，符号位不变，其余各个位取反.
   <div>
   $$
   \begin{array}{l}
   [+1] = [00000001]_原 = [00000001]_反 \\
   [-1] = [10000001]_原 = [11111110]_反
   \end{array}
   $$
   </div>


3. 补码的表示方法：
   正数的补码就是其本身
   负数的补码是在其原码的基础上，符号位不变，其余各位取反，最后 $+1$. (即在反码的基础上 $+1$)
   <div>
   $$
   \begin{array}{l}
   [+1] = [00000001]_原 = [00000001]_反 = [00000001]_补 \\
   [-1] = [10000001]_原 = [11111110]_反 = [11111111]_补
   \end{array}
   $$
   </div>


## 反码和补码的意义

我在上一篇文章[由 Math.abs 谈负数转换与绝对值运算]({{< ref "talk-about-negative-number-conversion-and-absolute-value-operation-through-math-abs" >}})中有提到补码的作用：

> 对于计算机来说加减是最基础的运算要设计的尽量简单，而减法相当于加上一个负数，所以完全可以使用加法来代替减法。但是这就出现了如何处理符号位的问题，short、int、long 的位数不同符号位自然也不同，为了避免运算前还需要判断符号位的位置，前人提出了将符号位也引入计算的机制，也就是补码。

确切的说，用反码就可以实现加法来代替减法， 其中唯一特殊的是 $0$：

<div>
$$
\begin{split}
&1 - 1 \\
&= 1 + (-1) \\
&= [0000 0001]_原 + [1000 0001]_原 \\
&= [0000 0001]_反 + [1111 1110]_反 \\
&= [1111 1111]_反 \\
&= [1000 0000]_原 \\
&-0
\end{split}
$$
</div>

这样算出来的结果是 $-0$，然而对于 $0$ 来说正负号是无意义的，并且 $-0$ 的表示方法使得 $0$ 出现了两个编码
此外，用反码加法代替减法需要循环进位：

<div>
$$
\begin{split}
&4 - 2 \\
&= 4 + (-2) \\
&= [0000 0100]_原 + [1000 0010]_原 \\
&= [0000 0100]_反 + [1111 1101]_反 \\
\end{split}
$$
</div>

这里如果按正常的二进制加法计算 $0000 0100 + 1111 1101$ 会得出 $0000 0001$ 即 $1$ 的错误结果。正确的做法是将溢出的进位再加到最低位上，得到正确结果 $0000 0010$ 即 $2$

为了解决 $0$ 有两个编码并且需要循环进位的问题，出现了补码:
补码用 $[00000000]_补$ 表示 0，而 $[10000000]_补$ 表示 $-128$，这样就消除了歧义

<div>
$$
\begin{split}
&1 - 1 \\
&= 1 + (-1) \\
&= [0000 0001]_原 + [1000 0001]_原 \\
&= [0000 0001]_补 + [1111 1111]_补 \\
&= [0000 0000]_补 \\
&= [0000 0000]_原 \\
&= 0
\end{split}
$$
</div>

综上所述反码和补码的意义在于可以让符号位参与计算，从而可以用加法代替减法的数学原理。接下来我们接下来我们探讨一下可以这样做背后的数学原理

## 用反码和补码的加法代替减法的数学原理

首先我们来介绍两个数学概念：

### 取模运算

取模运算公式：

$$ x \bmod y = x - y \lfloor x / y \rfloor $$

以 $ -3 \bmod 2 $ 为例:

<div>
$$
\begin{split} 
&-3 \bmod 2 \\
&= -3 - 2 \times \lfloor -3/2 \rfloor \\
&= -3 - 2 \times \lfloor -1.5 \rfloor \\
&= -3 - 2 \times (-2) \\
&= -3 + 4 \\
&= 1
\end{split}
$$
</div>

这里给出一个取模的算法实现：

```java
public int mod(int a, int b) {
    return (a % b + b) % b;
}
```

### 同余

两个整数 a，b，若它们除以整数 m 所得的余数相等，则称 a，b 对于模 m 同余

记作 $ a \equiv b \pmod m $，读作 a 与 b 关于模 m 同余

<div>
$$
\begin{split} 
\left. \begin{gathered}
4 \bmod 12 &= 4 \\
16 \bmod 12 &= 4 
\end{gathered} \right\}
\implies
4 \equiv 16 \pmod{12}
\end{split}
$$
</div>

同余具有的性质：

1. 自反性，对称性，传递性
<div>
$$
\begin{align}
&a \equiv a \pmod m \\
&a \equiv b \pmod m \iff b \equiv a \pmod m \\
&a \equiv b \pmod m \text{ 且 } b \equiv c \pmod m \implies a \equiv c \pmod m \\
\end{align}
$$
</div>

2. 线性运算
<div>
$$
\begin{split} 
a \equiv b \pmod m \text{ 且 } c \equiv d \pmod m \implies
\left\{ \begin{gathered}
a \pm c &\equiv b \pm d \pmod m \\
a \times c &\equiv b \times d \pmod m
\end{gathered} \right.
\end{split}
$$
</div>

{{< tip warning >}}
同余的除法的性质与加减乘法不同：
$$ a \equiv b \pmod m \quad 且 \quad c \equiv d \pmod m \implies a \equiv b\left(\bmod \frac{m}{\operatorname{gcd}(c, m)}\right) $$
{{< /tip >}}

3. 幂运算
   $$ a \equiv b \pmod m \implies a^{n} \equiv b^{n} \pmod m $$

### 证明

我们还是以 $ 4 - 2 $ 为例，根据同余数的性质我们很容易得出这样的结论：

<div>
$$
\begin{split}
\left. \begin{gathered}
4 &\equiv 4 \pmod{7} \\
-2 &\equiv 5 \pmod{7} \\
\end{gathered} \right\}
\implies 4 - 2 \equiv 4 + 5 \pmod{7}
\end{split}
$$
</div>

也就是说我们计算 $ 4 - 2 $ 就相当于计算 $ (4 + 5) \bmod 7 $
如果我们将 $-2$ 和 $5$ 换成二进制会发现 $-2$ 的反码的数值部分正好等于 $5$，也就是说补码的计算实际上是求其模数为当前数制的上限（这个例子中为 $7$）的同余数

<div>
$$
\begin{split}
-2 &= [1010]_原 = [1101]_反 \\
5 &= [0101]_原 = [0101]_反
\end{split}
$$
</div>



这样循环进位就相当于溢出的数对当前数制的上限（这个例子中为 $7$）取模后再与原结果相加

$$ (4 + 5) \bmod 7 = 8 \bmod 7 + 1 = 2 $$

这里的 $8$ 相当于进位

而补码的计算相比于反码只不过是增加了模的值：这时我们计算 $ 4 - 2 $ 就相当于计算 $ (4 + 6) \bmod 8 $

$$ (4 + 6) \bmod 8 = 8 \bmod 8 + 2 = 2 $$

因为进位 $ 8 \bmod 8 = 0 $ 所以不需要循环进位

{{< tip >}}
另一种数学推导可以看这篇文章：[反码与补码加法的理解](https://note.sbwcwso.com/pages/1dd33b)
{{< /tip >}}
