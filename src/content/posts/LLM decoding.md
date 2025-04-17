---
title: 大模型解码
published: 2025-04-17
draft: true
---

# 解码策略

## 基于sampling

sampling一般用在chat、writing这些需要有有一定创造力的任务上

greedy decoding：选取probability最大的token

pure sampling：按照每个token的probability选取token

tok-k：介于greedy decoding和pure sampling，greedy decoding其实可以看作top-k的k=1；pure sampling可以看作k=token-size。选取前k个probability最大的token，然后在这k个token里，按照每个token的probability选取token。

top-p：top-k中的k是一个需要人为指定的超参数，但是LLM进行decoding时候，每一步的distribution是不同的。对于probability distribution集中的情况，k选的太大有可能会取到分布tail上的token；对于probability distribution比较平，k选的太小有可能token的选择不够多样，很容易产生重复序列。top-p的思想就是要动态地产生一个k，在decoding过程中，如果每个token的probability从p1一直到pn，从最大的probability开始累加，计算cumulative probability，当累加的值大于设定的阈值P时，停止，选取这之前的k个token做采样。

Temperature 参数：对于一个probability distribution，可以使用Temperature改变它的分布形状，t = 1时不改变probability distribution形状，t < 1会将probability distribution压缩，将probability集中到更少的一些token上，t > 1时会讲probability distribution拉平，将probability分布到更多的token上

一般来说是先调节Temperature，再做samping

## 基于search

beam search一般用在translation、summary这些需要有稳定输出的任务上

Beam Search是一种启发式搜索算法，用于在每个时间步选择最可能的序列。它不是一次生成整个序列，而是逐步构建序列。以下是Beam Search的基本流程：

1. **初始化**：设定一个beam宽度（beam size），这个宽度决定了在每一步保留的候选序列数量。

2. **第一步选择**：在序列的第一个词上，根据模型预测的概率选择最高的几个词（数量等于beam宽度）作为初始候选序列。

3. **迭代构建**：对于每一步：
   - 对于每个候选序列，生成所有可能的下一个词，并计算整个序列的概率（通常是将概率相乘）。
   - 保留概率最高的几个序列（数量等于beam宽度），舍弃其余序列。
   - 如果达到预设的最大长度或遇到结束标记（如句号），则停止迭代。

4. **选择最终序列**：在所有终止的序列中，选择整体概率最高的序列作为最终输出。
