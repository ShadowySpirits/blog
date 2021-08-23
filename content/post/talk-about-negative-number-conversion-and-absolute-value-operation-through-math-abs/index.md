---
title: 由 Math.abs 谈负数转换与绝对值运算
slug: talk-about-negative-number-conversion-and-absolute-value-operation-through-math-abs
date: 2021-08-16T15:53:15+08:00
math: true
tags:
  - Java
  - Math
categories:
  - tech
---

本文通过分析一个 Java 中 `Math.abs()` 误用引发的 bug 介绍了计算机中数的储存、负数转换与绝对值运算

<!--more-->

## 背景

最近遇到了一个奇妙深刻的 bug：我们的系统中使用了一个 int 型的变量来计数，这个计数器变量的绝对值取模作为某个 list 的 index，但是今天出现了异常 IndexOutOfBoundsException

```java
// 满足某些条件计数器自增
int count = 0;
count++；

int index = Math.abs(count) % list.size();
// java.lang.IndexOutOfBoundsException: Index -2147483648 out of bounds for length 1
list.get(index);
```

## Math.abs 遇上 Integer.MIN_VALUE

我们知道 int 是用 32 位二进制储存的，其中最高位是符号位。上述代码不断自增 count 变量最终会使其超过上限从而反转成负数（`Integer.MAX_VALUE + 1 == Integer.MIN_VALUE` 会返回 true） 我们预料到了这种情况发生所以使用 `Math.abs(count)` 来确保取模的结果为正数，但是结果事与愿违

我们查看报错信息发现 -2147483648 正好是 Integer.MIN_VALUE，所以我们先试一下 `Math.abs(Integer.MIN_VALUE)` 结果居然是 -2147483648，这显然不符合我们的预期，让我们先来看一下 `Math.abs()` 的实现：

```java
    /**
     * Returns the absolute value of an {@code int} value.
     * If the argument is not negative, the argument is returned.
     * If the argument is negative, the negation of the argument is returned.
     *
     * <p>Note that if the argument is equal to the value of
     * {@link Integer#MIN_VALUE}, the most negative representable
     * {@code int} value, the result is that same value, which is
     * negative.
     *
     * @param   a   the argument whose absolute value is to be determined
     * @return  the absolute value of the argument.
     */
    @HotSpotIntrinsicCandidate
    public static int abs(int a) {
        return (a < 0) ? -a : a;
    }
```

注释中说明了如果 a 是 Integer.MIN_VALUE 那么返回值也是负数。到这里就破案了，只要把 `Math.abs(count) % list.size()` 改为 `(count % list.size() + list.size()) % list.size()` 即可，下面分析一下为什么会出现这种情况

{{< tip info >}}
改为 `(count % list.size() + list.size()) % list.size()` 而不是 `Math.abs(count % list.size())` 是因为前者符合取余的数学定义
详情可以看我的另一篇文章 [反码和补码的数学原理]({{< relref "the-mathematical-principle-of-ones-complement-and-twos-complement" >}})
{{< /tip >}}

## 运算符 - 的原理

分析 `Math.abs()` 的代码可以发现问题出在运算符 `-` 上，也就是说 `-Integer.MIN_VALUE == Integer.MIN_VALUE`，显然出现这种情况说明运算符 `-` 的作用不可能是简单的将符号位取反。

要继续分析就要先了解数在计算机中的储存原理：对于计算机来说加减是最基础的运算要设计的尽量简单，而减法相当于加上一个负数，所以完全可以使用加法来代替减法。但是这就出现了如何处理符号位的问题，short、int、long 的位数不同符号位自然也不同，为了避免运算前还需要判断符号位的位置，前人提出了将符号位也引入计算的机制，也就是补码。

补码的表示方法是:

- 正数的补码就是其本身
- 负数的补码是在其原码的基础上符号位不变，其余各位取反，最后 +1

计算机中计算 1 - 1 的原理：

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

所以运算符 `-` 的原理是进行求其补码的运算，即按位取反然后 +1，于是我们来分析 `-Integer.MIN_VALUE` 的运算过程

```java
-Integer.MIN_VALUE = ~Integer.MIN_VALUE + 1 = ~0x80000000 + 1 = 0x7fffffff + 1 = 0x80000000
```

可以发现在 `0x7fffffff + 1` 的过程中发生了溢出，所以最后的结果还是 0x80000000 也就是 Integer.MIN_VALUE

## 总结

通过以上分析我们可以知道，使用 `Math.abs()` 对 Short.MIN_VALUE、Integer.MIN_VALUE、Long.MIN_VALUE 取绝对值都得不到正确的结果。那有没有方法可以得到正确的结果呢？答案是 Integer.MIN_VALUE 的绝对值在 Integer 的范围内无法表示，因为 0 的存在 Integer. MAX_VALUE 比 Integer.MIN_VALUE 的绝对值小 1。如果确实要得到 Integer.MIN_VALUE 的绝对值可以先转换为 Long：`Math.abs(Long.valueOf(Integer.MIN_VALUE))`
