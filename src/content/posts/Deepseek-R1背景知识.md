---
title: 理解DeepSeek-R1的背景知识
published: 2025-04-17
draft: false
---

## RL

 为什么有要RL，sft存在的问题：

1. cost：需要大量的人力去label数据
2. not flexible：针对不同的任务，需要标注不同的数据或者对同一批数据进行不同的标注



【RL model的两种形式】

```
1. policy based (actor)		输出的是根据当前state，所有有可能的动作的概率，可以看作一个分类模型
2. value based 	(critic)	输出的是判断当前state的好坏，根据当前的state，所有有可能的情况得到的score的期望
```

在RL中，需要训练policy model或者value model或者两个都有

这里再插一句reward model和value model的区别，奖励模型评估即时奖励，强调当前行为的直接反馈；而价值模型估计长期回报，关注行为的未来影响。：

```
奖励模型（Reward Model）主要用于评估智能体在特定状态下执行某个动作所获得的即时奖励。它直接衡量某一行为的即时好坏，通常由环境提供。例如，在训练大型语言模型时，奖励模型可以通过人类反馈来学习，评估生成文本的质量，为后续的强化学习阶段提供准确的奖励信号 。

价值模型（Value Model）则关注从某一状态或状态-动作对开始，智能体在未来所能获得的累积奖励的期望值。它预测长期回报，帮助智能体评估当前行为对未来的影响，从而制定更优的策略。例如，在PPO（近端策略优化）算法中，价值模型（也称为Critic模型）用于估计当前策略下某一状态的价值，辅助策略的更新和优化 。
```



【RL】

**reward**

RL中reward的特点：

1. sparse，不同于sft中每一个数据都有一个label，RL中reward只在过程中某几步或最后才有reward
2. define，在sft中label往往比较明确，但是RL中并不是这样。在游戏、数学、编程等领域，结果比较明确，还是可以比较容易地定义reward；但是比如human perference，reward很难定义，在gpt3时代，是先让一批人来标注数据，根据这些标注数据训练了一个model，这个model是用于专门对LLM的输出进行打分，返回reward的，即这是一个reward model。后续的很多研究其实直接利用LLM来作为reward model了。但是这个过程需要提防reward hacking问题，即模型为了达到目的（获取最大的reward）而不择手段，类比一下，机器人为了达到世界和平的目的，就把世界上的人类全消灭了，因为机器人认为这样会永远世界和平。

**dymatically**

RL中有环境的概念，是可以与环境进行动态互动的

**expolre vs exploit**

Explore（探索）指尝试未知或未充分了解的选项，以获取更多信息或发现潜在的更好选择。在机器学习中，探索通常意味着尝试新的策略、模型参数或数据处理方式，以了解其效果。

Exploit（利用）指基于已有知识或经验，选择当前已知的最优或较优的选项，以最大化短期收益或性能。在实际应用中，利用通常是直接执行当前表现最好的方法。



【后续需要补充一下RL的发展历程 TRPO - PPO - DPO - GRPO】



## LLM training

### 训练范式

```
open-ai提出的LLM训练范式一般包含两个阶段：
    1. pre-train：用大量的text，使用NTP的训练方式进行预训练
    2. post-train：一般包含两个阶段
    	1）SFT
    	2）RL：做模型的alignment
```

大多数的LLM的训练基本还是沿用这个范式

![image-20250411150742270](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411150742270.png)

这里openai的RL过程类似如下流程：

1. 搞了一批人来写一些问题和答案，sft finetune model，使得模型先具有一些回复问题的能力
2. 收集一些数据，让模型回复这些问题，一个问题给出多个答案，然后让一批人来标注这些问题的好坏顺序，用标注好的数据来训练一个reward model
3. ppo的方法对模型进行RL



但是PPO实在太过于繁琐，而且reward model训练起来也很难（一是训练数据很难覆盖所有领域，二是即便数据搞定了，reward model还是很难很好的代表人的喜好，根据这样的reward model训练出来的模型效果可想而知），后续人们为了简便，提出了DPO

![image-20250411154443857](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411154443857.png)





### scaling law & emergent ability

![image-20250411152902531](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411152902531.png)

![image-20250411153000957](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411153000957.png)





## CoT 

### only inference

模型的思考能力大致可以分为两类

1. system 1，主要包括很多记忆的知识
2. system 2，主要是很多需要推理、思考的内容

![image-20250411164510491](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411164510491.png)



最初的CoT是在prompt中添加few-shot，使得模型模仿这种回复风格进行思考

![image-20250411170028321](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411170028321.png)

但是这种方式有一个很大的弊端是针对不同的任务需要人为地设计不同的CoT prompt，这是很繁琐的，所以有了后来的Zero-shot的CoT

![image-20250411170007266](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411170007266.png)

但是发现Zero-shot-CoT虽然很简单，但是整体性能是要差一点的，这个时候有人将上述两种方法结合试了一下

![image-20250411170816836](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411170816836.png)

后续又有人采用了多数投票的方法增强模型的性能，如下所示：

![image-20250411171433305](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411171433305.png)



### train

上述所有方法都是prompt 工程，只在推理阶段加入CoT，后续就有一些人把CoT也加入到了训练的部分，而将CoT加入训练部分，可以在post-train的两个步骤进行：1. sft. 2. RL.

#### sft

如何获取CoT的数据呢？这里主要是用LLM自己生成self-consistency的数据

![image-20250411171937651](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411171937651.png)

![image-20250411172208907](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411172208907.png)



#### RL

首先介绍一下RL在LLM中的范式

![image-20250411173256055](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411173256055.png)



reward model可以分为两种形式：

![image-20250411173420445](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250411173420445.png)

orm又可以分为两种：1. solution level的orm. 2. token level的orm.

prm则是对与思维链的每一个步骤都做验证，对每一个step做rewaed，在强化学习框架中可以提供更加丰富的监督信号



### CoT reward model

reward model本质是给定一个CoT，判断整体或者是每一步CoT的好坏。但是reward model其实是有两个应用场景，有不同的名字，如果reward model是放在RL框架下提供reward signal，则该reward model叫做reward model；如果reawrd model在test time compute时候使用，则该reward model叫做verifier。

#### verifier

在[openai的论文](https://arxiv.org/pdf/2110.14168)中提出了**orm**的方式验证模型最后的CoT是否正确从而实现test time compute

![image-20250417144037930](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250417144037930.png)

论文中的 generator 和 verifier都是gpt（gpt3 175B/6B）

generator是我们想要训练的大语言模型，希望它能产生CoT

verifier的作用是，给定一个CoT，判断该CoT是否正确

如何训练该verifier

```
1. 把训练数据中的Question和Solution（都是人工写的）一起送进大模型进行训练，本质是在做sft
2. 训练好后，给定一个Question，让大模型产生100个不同的Solution，然后让人工标注这100个Solution是否正确
3. 然后将产生的数据（Question，Solution，是否正确的标签）送进verifier做训练，本质也是在做sft
	verifer的作用是，给定一个（Question，Solution）判断该CoT是否正确
```

在Inference时候如何使用呢？其实这也就是openai首次提出的test time compute：

给定一个问题Q，首先让generator（gpt）产生100个带CoT的答案（Solution），然后每个答案经过verifier并预测其是否正确。针对每个答案，verifier都给定一个score，然后从中选出最好的答案。因为大部分的计算都发生在inference time，所以称为test time compute。该方法与前面only inference小节中提到的self consistent的方法非常像，唯一的区别是引入了verifier，而不是使用majority voting的方法。

具体的训练细节是使用**联合目标函数（joint objective）**来训练 verifier（验证器），使得模型在完成原始语言建模任务的同时，还能学习判断模型生成结果是正确还是错误。verifier 本身就是一个语言模型，但在其输出层上加了一个小型的**标量头（scalar head）**，该头部可以为每个 token 给出一个预测结果。用一个**偏置参数（bias）**和一个**增益参数（gain）**来实现这个标量头，它们作用于语言模型最终反嵌入层（unembedding layer）输出的 logits 上。这两个参数会对词表中某个**特殊 token**对应的 logit 值进行平移（加偏置）和缩放（乘增益）。其他 token 的 logits 仍用于完成语言建模任务，而这个特殊 token 的 logit 就被专门用于验证器的预测任务。我们可以选择用与生成器相同的预训练语言模型来初始化 verifier，也可以直接从生成器本身进行初始化。在消融实验中，后者（从生成器初始化）表现略优。我们认为这是因为：若 verifier 更好地理解了生成器学习到的语言分布，那么它对该分布下样本的评分会更准确。

![image-20250417153500310](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250417153500310.png)

可以看到右图中绿色的是对每个token的prediction，后续的实验证实，token level的verifier也是更好的。

后续openai又发了[一篇paper](https://arxiv.org/pdf/2305.20050)，主要是prm model，并且证明了说prm model是比orm model效果要更好

![image-20250417155946517](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250417155946517.png)

有了这样一个针对每个step标注的数据集，可以训练prm model，类似orm model的训练策略，就可以给定每一个推理步骤都判断是对或错

![image-20250417160234771](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250417160234771.png)

有了这样的prm model后，在inference时，针对每个问题Q，会产生很多的答案Solution，每个Solution都有很多的步骤step，prm会给每个step都输出一个probability，将每一步的probability乘起来，作为整个Solution的打分，并用于筛选答案。

![image-20250417160646896](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250417160646896.png)

#### reward signal

此外，[deep-mind也有一篇文章在讨论prm和orm](https://arxiv.org/pdf/2211.14275)，并且不同的是，这篇文章不仅将prm和orm用到了RL框架中作为reward signal的来源，也在test time compute的时候用作选取最佳答案的verifier

![image-20250417161315978](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250417161315978.png)

这里[deepseek也有一篇文章](https://arxiv.org/pdf/2312.08935)，提出了一种自动获取prm训练过程中每一个step需要的label

![image-20250417162453887](https://lien-bucket.oss-cn-shenzhen.aliyuncs.com/lien-bucket.oss-cn-shenzhen.aliyuncs.comimage-20250417162453887.png)

