---
title: Counting Large Numbers of Events in Small Registers (1978)
date: 2021-01-24 14:40:16
tags:
- papers-we-love
mathjax: true
---

最近开始听 Harvard 的 CS 299r，即 "Algorithms for Big Data" 这门课。第一节课除了介绍课程大纲外，还介绍了近似计数问题 (Approximate Counting Problem)，而其主要内容来自于 1978 年 Robert Morris 发表的这篇短文。本文尝试将课程和论文结合做一个小结。

>  注：由于概率论知识储备有限，本文个别地方尚不严谨，希望他日能给出详细证明。

## Approximate Counting Problem

通常，一个 k-bit 寄存器的计数上限是 $2^k - 1$。由于种种限制，Morris 面对的场景限制如下：

* 无法扩展寄存器长度 ($k = 8$)
* 需要计数的事件总数远超过 255
* 计数结果无需精确，但相对误差 (relative error) 固定

我们将这个问题进一步提炼，可以重述为：**使用最小的空间统计事件发生的总数**，且对于估计的结果 $\tilde{n}$，满足：
$$
\begin{align*}
\mathbb{P}(|\tilde{n} - n| > \varepsilon n) < \delta
\end{align*}
$$
这里的 $\varepsilon = \frac{|\tilde{n} - n|}{n}$ 即为相对误差。

## 首次尝试

为了使得计数范围超过精确计数的空间限制，最简单地方式就是**每 $s$ 个事件计数值加 1，即每次遇到新的事件，按$\frac{1}{s}$ 的概率自增**。这是经典的多重伯努利实验，其分布符合二项分布 (Binomial distribution)，二项分布 $B(n, p)$ 的均值 $E(x) = np$，方差 $\sigma^2 = np(1-p)$，那么其相对误差 $\epsilon = \sqrt{\frac{p(1-p)}{n}}$，其中 $n$ 越小，相对误差越大；$n$ 越大，相对误差越小。这并不符合相对误差固定的要求，因此该方案不符合要求。

## 算法空间复杂度下限

假设我们有一个确定性 (deterministic) 算法实现近似计数，这个算法的空间复杂度下限是多少？近似意味着当实际总数落在某个区间 $[n_1, n_2]$ 中时，给出的估计值为 $\tilde{n_1}$。假设我们有 $k$ bits 空间，那么在该空间中最多可以表示 $2^{k}$ 个区间。我们将这些区间标记为 $a_1$、$a_2$、...、$a_{2^k}$。为了在保证固定相对误差的同时尽量扩大计数范围，区间大小应该呈指数增长。假设这个指数为 2，即 $a_{k+1} = 2a_k$，那么可以推测这些区间最多可以表示约 $2^{k^k}$ 个数。换句话说，假设计数上限为 $n$，该计数方案的空间复杂度为 $O(log(logn))$。

> 注：以上推理过程仅作为直觉参考，非严格证明

举一个具体的例子，假设我们将事件总数取值为最近的一个 $2^k, k=0,1,2,...$：

| 数值     | 1    | 2    | 3             | 4    | 5             | 6             | 7             | 8    | 9             | 10            | 11             | 12            | 13             | 14            | 15             | 16   |
| -------- | ---- | ---- | ------------- | ---- | ------------- | ------------- | ------------- | ---- | ------------- | ------------- | -------------- | ------------- | -------------- | ------------- | -------------- | ---- |
| 估计值   | 1    | 2    | 4             | 4    | 4             | 8             | 8             | 8    | 8             | 8             | 8              | 16            | 16             | 16            | 16             | 16   |
| 相对误差 | 0    | 0    | $\frac{1}{3}$ | 0    | $\frac{1}{5}$ | $\frac{1}{3}$ | $\frac{1}{7}$ | 0    | $\frac{1}{9}$ | $\frac{1}{5}$ | $\frac{3}{11}$ | $\frac{1}{3}$ | $\frac{3}{13}$ | $\frac{1}{7}$ | $\frac{1}{15}$ | 1    |

不难看出，该算法的的误差始终小于 $\frac{1}{3}n$，即相对误差固定。但实际上并不存在这样一个算法，因为它要求我们必须事先知道当前的事件总数 $n$，才能计算出实际的存储值 $\tilde{n}$，但既然我们已经知道事件总数，再计算估计值就没有意义了。

## Morris

如果计数器存储的值为 $v$，假设存在满足 $v(n) = log_{2}(1 + n)$ 的算法。若 $v$ 的当前值为 $v_k$ (这里 $v_k$ 是算法能知道的全部信息)，在计数器收到一个新的事件之前，已发生事件总数的估计为：
$$
n_{v_k} = 2^{v_k} - 1
$$
加上最新的事件后事件总数为 $2^{v_k}$，那么下一次存入计数器的值应该为：
$$
v_{k+1} = log_2(1 + 2^{v_k})
$$


但显然这不是个整数。我们可以利用概率，设 $\Delta = \frac{1}{n_{r+1} - n_{r}}$，即 $\frac{1}{2^{v_k}}$：
$$
v_{k+1} = 
\begin{cases} 
      v_{k} & p = 1 - \Delta \\
      v_{k} + 1 & p = \Delta
\end{cases}
$$
就能满足预期。现在正式地描述 Morris 近似计数算法：

1. 初始化变量 $X \leftarrow 0$
2. 每出现一个事件，以概率 $\frac{1}{2^X}$ 为 $X$ 增加计数 1
3. 输出近似计数值 $\tilde{n} = 2^X - 1$

利用数学归纳法及 Chebyshev 不等式，分别可以证明：
$$
\mathbb{E}2^{X_{n+1}} = n + 1
$$
以及：
$$
\mathbb{P}(|\tilde{n} - n| > \varepsilon n) < \frac{1}{2\varepsilon^2}
$$
详细证明见 [lecture 1 scribe](http://people.seas.harvard.edu/~minilek/cs229r/fall15/lec/lec1.pdf)。尽管相对误差固定，但其相对误差小于 $\varepsilon$ 的概率上限过大，不具备实践意义。

## General Morris

上文中的 Morris 基于 $v(n) = log_{2}(1 + n)$，将该公式泛化可以得到：
$$
v(n) = \frac{log(1 + \frac{n}{a})}{log(1 + \frac{1}{a})}
$$
这里的分母 $log(1 + \frac{1}{a})$ 仅用于保证当 $n = 1$ 时，$v=1$，当实际计数为 0 或 1 时结果精确，这时可以表示的最大计数值为：
$$
n_v = a((1 + \frac{1}{a})^v - 1)
$$
在 $n$ 次事件发生后的计数的方差为：
$$
\sigma^2 = \frac{n(n-1)}{2a}
$$
取 $a=30$，其最大的能表示的数值约为 130000，其标准差约为 $\frac{n}{8}$，这意味着相对误差约为 $\frac{1}{8}$，既相对误差小于 24% 的概率大于 95%。这使得 Morris 具备了实践意义。

> 注：这里应该可以给出类似上一节中 $\mathbb{P}(|\tilde{n} - n| > \varepsilon n)$ 与 $a$ 的关系，但论文和课程中都没有给出相关证明，我暂时没有扎实的概率论基础及欲望去证明，如果读者有兴趣可以试试。

这里的直觉可以理解为：$a$ 越大，可以近似估计的最大值约小，每个值对应的区间越小，相对误差也越小。

## Morris+

假设我们的目标是 $\varepsilon = \frac{1}{3}$，$\delta = 0.01$，基本版 Morris 并无法满足这样的要求。如果所用的空间允许线性扩展，即保持空间复杂度 $O(loglog(n))$ 不变，我们可以初始化 s 个 Morris 计数器副本，取它们的平均值。三个臭皮匠赛过诸葛亮，这时 Morris 一节中最后关于 $\mathbb{P}(|\tilde{n} - n| > \varepsilon n)$ 的公式可以转化为：
$$
\mathbb{P}(|\tilde{n} - n| > \varepsilon n) < \frac{1}{2s\varepsilon^2}
$$
那么理论上我们可以通过增加副本数量无限降低概率上限，从而满足要求。

## Morris++

还有一种更省空间的方式，就是使用 $t$ 个失效概率为 $\frac{1}{3}$ 的 Morris+ 计数器副本，即 $s = \theta(\frac{1}{\varepsilon^2})$，从 $t$ 个 Morris+ 副本中选择中位数作为最终估计值，当 $t = \theta(lg\frac{1}{\delta})$ 时，$\mathbb{P}(|\tilde{n} - n| > \varepsilon n) < \delta$。详细证明见  [lecture 1 scribe](http://people.seas.harvard.edu/~minilek/cs229r/fall15/lec/lec1.pdf)。

## Takeaways

* 近似估计的基本思路就是通过将精确数值转化为区间估计值
* 理解算法复杂度下限帮助我们知道可能性边界在哪。就如基于比较的排序算法的复杂度下限为 $O(nlogn)$。
* 多个副本取均值、中位数的方式是基于统计的估计算法的常用优化方式

## References

* [Counting Large Numbers of Events in Small Registers](https://www.inf.ed.ac.uk/teaching/courses/exc/reading/morris.pdf)
* CS 229r: Algorithms for Big Data (2015), [syllabus](http://people.seas.harvard.edu/~minilek/cs229r/fall15/lec.html), Lecture 1 [scribe](http://people.seas.harvard.edu/~minilek/cs229r/fall15/lec/lec1.pdf), [video](https://www.youtube.com/watch?v=s9xSfIw83tk&index=1&list=PL2SOU6wwxB0v1kQTpqpuu5kEJo2i-iUyf)

