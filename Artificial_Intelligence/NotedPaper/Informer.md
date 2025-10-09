# Informer: Beyond Efficient Transformer for Long Sequence Time-Series Forecasting

## Abstract

>**LSTF**,Long sequence time-series forecasting

Transformer架构的出现给长时间序列预测带来了新的潜力。然而仍有以下几个严重问题：平方时间复杂度、高内存使用、以及encoder-decoder架构的自身限制。

本文设计了一个基于transformer架构的高效LSTF模型，称*Informer*。

1. ProbSparse 自注意力机制
   实现了 $O(L logL)$ 级别的时间复杂度与内存使用以及序列依赖对齐。
2. 自注意力蒸馏
   削减瀑布层输入，关注高价值注意力
3. 一步式decoder
   取代step-by-step方式，采用one forward operation优化推理速度

## Introduction

时间序列预测的多数方法都是基于短期预测问题而设置的。越来越长的时间序列对现有方法是个挑战。

>The prediction capacity of existing methods limits LSTF’s performance. E.g., starting from length=48, MSE rises unacceptably high, and the inference speed drops rapidly.

LSTF的主要挑战是，加强模型预测能力来满足越来越长的序列要求，这需要更强的长距离对齐能力与处理长序列的高效方式。

transformer架构有比传统RNN方法更好的表现，自注意力机制将signal的传播路径压缩到理论上的最短 $O(1)$ 事件，并避免了循环结构，使Transformer在LSTF问题上表现出巨大潜力。

然而其时间复杂度与训练、部署的花费使其在LSTF问题上令人难以承担。一个高效的自注意力机制和transformer架构成了当前瓶颈。

本文提出的问题是：是否可以改进Transformer架构优化其计算、内存与架构性能，并同时维持一个高预测准确率。

---

多数改善自注意力机制的方案都主要聚焦于削减时间复杂度。本文探究了自注意力机制的稀疏性，改善了网络组建，贡献总结如下：

1. 提出 **Informer** 成功的增强了预测能力，并确认了类transformer模型在长时间序列上捕捉长距离独立依赖上的潜力。
2. 提出 **ProbSparse** 自注意力机制取代传统自注意力机制，在依赖对齐上实现了 $O(LlogL)$ 的时间复杂度与内存使用。
3. 提出 自注意力蒸馏操作 来强调主导注意力得分，将空间复杂度降低到 $O((2-?)LlogL)$
4. 提出生成式decoder 用一步前向操作获得长序列输出，同时避免推理阶段误差累计的扩散。

![alt text](./resources/Informer/image.png)

>Informer架构总览。
>左侧的encoder接受大量长序列输入(绿色序列)我们将原始的自注意力机制替换为提出的ProbSparse自注意力
>蓝色四边形是自注意力蒸馏操作，提取主导注意力，显著减少网络大小。层的堆叠以增加鲁棒性
>右侧的decoder接受长序列输入，将目标元素填充为0，测量特征图的加权注意力组合，以生成式风格预测输出元素(橙色)