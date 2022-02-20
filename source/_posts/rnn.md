---
title: 从头开始实现 RNN
date: 2022-02-20 15:03:20
tags:
---

> What I cannot create, I do not understand. -- Richard Feynman

Andrej Karpathy 在 2015 年发表了题为 [The Unreasonable Effectiveness of Recurrent Neural Networks](https://karpathy.github.io/2015/05/21/rnn-effectiveness/) 的博客，并配套开源其中实验所用的[char-rnn 代码仓库](https://github.com/karpathy/char-rnn)，以及用 numpy 手写的 [gist: min-char-rnn](https://gist.github.com/karpathy/d4dee566867f8291f086)，阅读过后受益良多。于是这两天花了些时间：

1. 逐行理解 min-char-rnn
2. 基于理解和原脚本实现了 2-layer 和 n-layer 的 RNN

整个探索过程充满了趣味和挑战，尤其是对于一位主营服务端开发的工程师，因此特意将这个过程记录下来。

<!-- more -->

> ⚠️ 本文假设读者有一定的深度学习理论基础和实践经验，同时阅读过上述的博客和脚本。但如果不介意可能出现的理解障碍，也欢迎继续阅读本文。

## 理解 min-char-rnn

min-char-rnn 实现的是单层的 Vanilla RNN，其整体结构如下图所示：

![The architecture of min-char-rnn](./one-layer.png)

> 💡 一位热心网友就着 min-char-rnn 给出了一个带注释的[版本](https://github.com/eliben/deep-learning-samples/blob/master/min-char-rnn/min-char-rnn.py)，阅读后者的难度会更小一些

### 前向传播

以下是源码中的前向传播部分：

```python
for t in xrange(len(inputs)):
    xs[t] = np.zeros((vocab_size,1)) # encode in 1-of-k representation
    xs[t][inputs[t]] = 1
    hs[t] = np.tanh(np.dot(Wxh, xs[t]) + np.dot(Whh, hs[t-1]) + bh) # hidden state
    ys[t] = np.dot(Why, hs[t]) + by # unnormalized log probabilities for next chars
    ps[t] = np.exp(ys[t]) / np.sum(np.exp(ys[t])) # probabilities for next chars
    loss += -np.log(ps[t][targets[t],0]) # softmax (cross-entropy loss)
```

通常比较巧妙的地方就是令人费解的地方，这里比较巧妙的自然是 CrossEntropy 的计算：

```python
loss += -np.log(ps[t][targets[t],0])
```

CrossEntropy 计算的是两个概率分布的差异程度，差异越大，熵值越大。这里的两个概率分布分别是「RNN 预测的下一个字符的概率分布」和「实际下一个字符的概率分布」，在监督学习过程中，后者是一件已经确定的事情，所以它的概率分布是 [0, 0, ..., 1, 0, 0]，即除一个确定字符的出现概率为 1 外，剩余字符的出现概率皆为 0。于是最终对损失函数产生贡献的只有一个字符出现的概率，即这里的 `ps[t][targets[t], 0]`。

### 反向传播

以下是源码中的反向传播部分：

```python
for t in reversed(xrange(len(inputs))):
    dy = np.copy(ps[t])
    dy[targets[t]] -= 1 # backprop into y. see http://cs231n.github.io/neural-networks-case-study/#grad if confused here
    dWhy += np.dot(dy, hs[t].T)
    dby += dy
    dh = np.dot(Why.T, dy) + dhnext # backprop into h
    dhraw = (1 - hs[t] * hs[t]) * dh # backprop through tanh nonlinearity
    dbh += dhraw
    dWxh += np.dot(dhraw, xs[t].T)
    dWhh += np.dot(dhraw, hs[t-1].T)
    dhnext = np.dot(Whh.T, dhraw)
```

如果你像我一样只对数学分析中最基本的「单变量求导」还留存有一定的记忆，那看这段代码会有种「表面上好像能看懂，仔细思考又觉得哪里对不上」的感觉。具体地说，我们很容易看出来这些代码是在将损失函数的变化按相反的方向一步一步地倒推回去，再加上脑海中尚存的单变量「链式求导」规则，似乎一切顺理成章。但细思极「恐」，这里面有标量、有向量，还有矩阵，什么是「标量对矩阵求导」？什么是「向量对向量求导」？什么是「矩阵对向量求导」？什么是「矩阵对矩阵求导」？我们学过「向量求导」或者「矩阵求导」吗？

为了消解脑海中尚不清晰的地方，我在搜索引擎中以类似「matrix calculus for engineer」的关键词，找到了一篇长文：[The Matrix Calculus You Need For Deep Learning](https://explained.ai/matrix-calculus/index.html)，它正好是为像我这样**只记得「单变量求导」的人**量身定做的资料，于是我集中精力花了 3-4 个小时把全文通读，随后在笔记本上将代码中的几个标量 (损失函数)、向量、矩阵间的求导都推了一遍，终于云开雾散，事实证明这样的付出十分值得。

Andrej 在反向传播 Softmax + CrossEntropy 时写了个备注：

> backprop into y. see http://cs231n.github.io/neural-networks-case-study/#grad if confused here

令人沮丧的是，当你仔细阅读这部分内容，会发现这样一句话：

> It’s a fun exercise to the reader to use the chain rule to derive the gradient, but it turns out to be extremely simple and interpretible in the end, after a lot of things cancel out...

真正看懂它，是我在阅读完上面那篇关于 Matrix Calculus 的扫盲文章，紧接着继续阅读这篇博客 [The Softmax function and its derivative](https://eli.thegreenplace.net/2016/the-softmax-function-and-its-derivative/) 之后。**最终的答案很具有美感，但推导的过程很需要耐心**。

值得一提的是，这里在训练的时候还使用了「teacher forcing」的技巧：

```python
dy = np.copy(ps[t])
dy[targets[t]] -= 1
```

即不管 RNN 在每次 unroll 都使用标准答案，而不是自身学到的结果 `np.argmax(ps[t])`，在其它教程中，我也看到过人们会给「teacher forcing」施加一个概率，就像 dropout_p，因为在推理过程中不会有任何字符的参考结果，据说通过这个概率可以一定程度上避免「学生在实际解题时过分依赖老师」。

### Gradient Check

在 min-char-rnn 脚本下的第一个评论，就是 Andrej 自己的：

![gradient check](./gradient-check.png)

刚看到这段代码，我完全不知道 gradient check 是在做什么，自然无法理解这段代码的含义。不过在搜索引擎的帮助下，找到了斯坦福大学开放的教程 [Unsupervised Feature Learning and Deep Learning](http://ufldl.stanford.edu/tutorial/)，其中一节正是介绍 [Debugging: Gradient Checking](http://ufldl.stanford.edu/tutorial/supervised/DebuggingGradientChecking/)。所谓 gradient check(ing) 其实就是对比「通过反向传播计算出来的梯度」与「利用导数定义近似计算得到的梯度」，来判断自己的反向传播阶段代码是否写对了，是工程师构建神经网络时常用的一种简单有效的 debug 方法。由于每次训练都可能非常耗时，而且一些 bug 即便存在，表面上也可能看着一切正常，在构建完神经网络后，开始训练之前，执行一下 gradient check(ing) 很有必要。在实现 2-layer 和 n-layer RNN 的过程中，gradient check(ing) 就成功帮助我发现了代码中的若干逻辑问题，这样的逻辑问题通过传统的「print 调试」、「单点调试」、「眼神调试」都极难发现。

## 实现 2-layer 和 n-layer RNN

2-layer 的 RNN 结构如下图所示：

![The architecture of a 2-layer RNN](./two-layer.png)

源码可以在[这里](https://github.com/ZhengHe-MD/replay-nn-tutorials/blob/main/min-char-rnn/min_char_rnn_two_layers.py)找到，其中的变量命名与上图的结构一致。

n-layer 的 RNN 结构如下图所示：

![The architecture of a n-layer RNN](./n-layer.png)

源码可以在[这里](https://github.com/ZhengHe-MD/replay-nn-tutorials/blob/main/min-char-rnn/min_char_rnn_n_layers.py)找到，其中的变量命名与上图的结构一致。

## 尾声

后续我将继续在工作之余，尝试裸写 LSTM 和 GRU，然后逐步复现 Andrej 在七年前完成的其它实验 : )。

## 参考资料

* [The Unreasonable Effectiveness of Recurrent Neural Networks](https://karpathy.github.io/2015/05/21/rnn-effectiveness/)
* [Github: karpathy](https://github.com/karpathy), [char-rnn](https://github.com/karpathy/char-rnn), [min-char-rnn](https://gist.github.com/karpathy/d4dee566867f8291f086)
* [The Matrix Calculus You Need For Deep Learning](https://explained.ai/matrix-calculus/index.html)
* [The Softmax function and its derivative](https://eli.thegreenplace.net/2016/the-softmax-function-and-its-derivative/)
* [UFLDL Tutorial](http://ufldl.stanford.edu/tutorial/), [Debugging: Gradient Checking](http://ufldl.stanford.edu/tutorial/supervised/DebuggingGradientChecking/)
* [Neural Networks and Deep Learning](http://neuralnetworksanddeeplearning.com/index.html)
* [eliben/deep-learning-samples/min-char-rnn](https://github.com/eliben/deep-learning-samples/blob/master/min-char-rnn/min-char-rnn.py)
* [ZhengHe-MD/replay-nn-tutorials/min-char-rnn](https://github.com/ZhengHe-MD/replay-nn-tutorials/tree/main/min-char-rnn)
