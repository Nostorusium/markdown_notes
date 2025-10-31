# 嵌入模型

一段文字是如何转换成一个embedding向量的？

## pooling

一句话中的每个token被转化为embedding向量后，对它们进行**池化**pooling。

> **pooling**指将一整段向量压缩成一个更短、或单一向量的过程。即把一整串token的信息汇聚成一句话的整体特征。
> 常见的pooling有:
>  
> 1. mean pooling,平均所有词向量
> 2. max pooling,取每个维度上的最大值
> 3. CLS pooling,用[CLS]标记的向量代表整个句子
> 4. attention pooling 有权重的平均

RAG中、绝大多数的开源的embedding模型都是基于sentence-bert的。因为bert基于encoder，能更好地理解前后文。句子被投入bert,进行pooling后，两个句子进行余弦相似度计算得到损失函数进行训练。

## 选取

1. Sequence Length
   大就是好
2. Embedding Dimensions
   并不是越大越好。如果业务场景的语义特别丰富，那么越大越好。如果比较垂直领域，选择小的反而更好。
3. 对比实验嵌入效果