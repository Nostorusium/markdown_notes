# Inference

## The Two-Phase Inference Process

推理包含两个阶段：the Prefill Phase 预填充阶段和the Decode Phase 解码阶段。

---

**prefill**阶段包括：

1. tokenization
   将输入文本转化为token
2. embedding conversion
   将token进行嵌入
3. initial processing
   将嵌入向量送进神经网络，创建丰富的关联信息。

prefill阶段是计算密集的，他需要一次性处理完所有的输入token。

---

**decode**阶段包括：
1. **Attention Computation**
   让token之间进行注意力运算。
2. **Probability Calculation**
   计算出下一token的似然概率
3. **Token Clection**
   基于似然概率，选择下一个token生成
4. **Continuation Check**
   决定要不要继续生成

decode阶段是内存密集型的，因为模型需要追踪所有已生成的token。

## Sampling Strategies 采样策略

当模型选择下一个生成的token时，他会先从词汇表中每个词的原始概率 *raw probability*，也叫 *logit* 开始。要最终决定选用哪个词生成仍可分为几个步骤：

- **Raw Logits**
  考虑词的原始概率
- **Temperature Control**
  更高的T意味着选择更加随机，更有创造性。
  更低的T意味着更准确。
- **Top-p(Nucleus) Sampling**
  比起考虑所有可能的词，我们只关注最有可能的几个选择。我们只选几个词进行概率累加，直到抵达一个概率阈值(比如top 90%)
- **Top-k Flitering**
  一个替代的方法，只考虑前k个最可能的词。

## Mananging Repetition 管理重复性

LLM的一个问题是他们倾向于生成重复内容。我们提出两种惩罚策略：

- **Presence Penalty** 存在惩罚
  一个固定的惩罚，对所有先前出现过的token施加，不管出现频率。
- **Frequency Penalty** 频率惩罚
  一个可放缩的惩罚，基于一个token的使用频率。频率越高，越不可能被选中。

惩罚最终调整raw probability，先于采样策略之前。

## Controlling Generation Length:Setting Bondaries

以下方式都可以限制生成长度:
- **Token Limits**
  设置最短最长token数量
- **Stop Sequences**
  明确结束生成的对话pattern
- **End-of-Sequence Detection**
  让模型自然的生成EOS

## Beam Search: Looking Ahead for Better Coherence 集束搜索

集束搜索 *Bean Search* 是一个更加全局的方法，不再每次只做一个选择，它同时探索多种可能的路径。工作原理如下：
1. 每步维持几个候选句(典型地，5~10个)
2. 对于每个候选句，计算下一token的可能性
3. 只选择最佳的词句组合
4. 继续这个过程，直到抵达某个长度阈值或者停止条件
5. 选用全局概率最高的句子

比起简单方法，这需要更多计算资源。

## Practical Challenges and Optimization

有以下几个关键指标供具体实现参考：
- **Time to First Token**(TTFT)
  获得第一个响应的速度。对于用户体验很重要，主要由prefill阶段决定。
- **Time Per Output Token**(TPOT)
  生成下一token的速度有多快。这决定了全局生成速度。
- **Throughput** 吞吐量
  能同时处理多少个请求。这影响扩展能力和开销。
- **VRAM Usage**
  占用了多少GPU内存。是现实世界中应用的主要限制。

---

上下文长度也是一个限制。
- Memory Usage
  随着上下文长度平方增长
- Processing Speed
  随长度变慢。
- Resource Allocation
  需要仔细地平衡VRAM使用

---

为了处理这些挑战，一个强而有力的优化方法是**KV缓存**，*KV(Key-Value) caching*
这项技术显著的提高了推理速度，通过存储和复用中间计算结果。

代价是需要额外的内存，但其性能提升足以弥补开销。