# Maximum Likelihood Estimation

最大似然估计来源于统计学的一个基本原理：寻找使观察到的数据出现概率最大的参数值。

$L(\theta)$ 表示在参数 $\theta$ 下观测到的当前数据的概率。
一般而言我们使用对数将乘积化为 $log$ 的累加。
$$
log L(\theta) = \sum_{i=1}^N logP(y_i|x_i;\theta)
$$

其中，这个条件概率的含义是：在给定输入 $x_i$ 和参数 $\theta$ 的条件下，输出为 $y_i$ 的概率。
则损失函数为:

$$
Loss(\theta) = -logL(\theta) = -\sum_{i=1}^N logP(y_i|x_i;\theta)
$$

交叉熵损失函数如下：

$$
Loss = -\sum_{i=1}^n y_i log(p_i)
$$

其中 $y_i$ 为one-hot编码后的真实分布。
在数学上，这两个东西等价。证明略。