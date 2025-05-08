---
title: DeepSeek-R1的一些理解
published: 2025-04-18
draft: false
---

## R1

### 数据

可以通过[这篇文章](https://arxiv.org/pdf/2306.01116)了解一下LLM在准备数据时候是一个怎么样的流程 

![image-20250417163724129](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417163724129.png)

其中deduplication是非常重要的，[google有一篇文章](https://arxiv.org/pdf/2107.06499)专门讲如何做deduplication，这里不再详细展开。如果没有做到很好的去重，并在这样的数据集上做了训练，会产生什么影响呢？这里也有一篇[文章](https://arxiv.org/pdf/2310.10226)，从数据的角度讨论了一下neural text degeneration产生的原因。

![image-20250417164436401](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417164436401.png)



### Deepseek-R1

首先看一下R1这篇论文的测试结果

![image-20250410153227904](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250410153227904.png)

需要对这些数据集有一个比较清楚的认识：

```text
数学：
	AIME 2024：高中水平
	MATH-500 ：研究生水平
代码：
	Codeforces：与真人程序员进行PK，指标96表示超越了96%的程序员
	SWE-bench Verified：根据github的issue解决问题，代表模型debug的能力
知识：
	MMLU：各个领域的常识，表示人类平均水平
	GPQA Diamond：题目少，但是非常难，phd水平
```



下面介绍R1的三个主要工作

![image-20250417180208332](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417180208332.png)



#### DeepSeek-R1-Zero

这是一个纯用RL训练出的模型

![image-20250410155133335](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250410155133335.png)

虽然也有不错的效果，但是也表现出一些问题，比如可读性很差（有些问题人类不可读）和中英文混杂输出（这两个问题并不是一个问题）

其中用到的reward有两个，accuracy和format

![image-20250417212228324](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417212228324.png)

aha moment：

![image-20250417211419197](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417211419197.png)

 

#### DeepSeek-R1

为了解决上述问题，DeepSeek训练策略上做了一些改进：

```
round one：
	目的是增强model reasoning 能力
    1. sft initialization：先用少量的cold-start data做了在DeepSeek-V3-Base model上做了finetune
    2. 做RL
round two：
	做bootstrap（想象自己提起自己的鞋带把自己拉上天），目的是增强模型的reasoning和general能力
	1.sft：用round one中2.得到的模型产出一批高质量数据，然后与1.中做sft的数据混合，重新对DeepSeek-V3-Base model做finetune
	2.做RL
```

![image-20250410155533434](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250410155533434.png)

##### round one

对于sft initialization：需要有sft data（这里论文中叫做cold start data） --> sft

```
这些cold start data如何获取呢？
1. few-shot long CoT as an example，在prompt中加入few-shot，这些示例包含CoT思考过程，加入这些示例后，模型在回复时就会自己产生一些CoT
2. directly prompt models to generate detailed answers，类似zero-shot的方法，比如加入“Let's think step by step”
3. R1-zero产生的数据 + human post-processing
```

做完sft就可以进行RL，这里RL用到的reward策略还是GRPO，和R1-zero相同，但是reward signal不同，有三个：accuracy reward、format reward和language consistent reward（主要是为了解决R1-zero的中英文混杂问题）。



##### round two

这轮想要增强是模型的reasoning和general能力

```
round two的sft：
	sft data --> sft(并没有在round one的checkpoint上继续finetune，而是在deepseek v3上finetune)
	sft data（reasoning data + non-reasoning data）：
	1. reasoning data（共600K）来自round one训练好的模型产生大量的数据，然后做rejection sampling（一部分用规则评估（如数学答案格式），一部分用 DeepSeek-V3 打分，并过滤掉混乱语言/代码等低质量样本）
	2. non-reasoning data（共200K）来自：1）deepseekv3做sft时的data，2）对v3 通过加入CoT prompt的方式产生新的数据，简单的任务不提供CoT提示
	
做完sft后可以开始做RL

round two的RL:
	这里的RL和instruct gpt中的方式类似
	reward有：
		1. rule based --> reasoning能力
		2. human perference	--> general能力
	还用了diverse prompt增强模型的泛化能力

```

对于上述rejection sampling的定义：拒绝采样是一种从复杂分布中采样的方法，通过对生成的数据进行筛选，只保留符合特定条件的高质量样本。在机器学习中，特别是在生成模型或强化学习场景中，拒绝采样常用于从模型生成的输出中挑选出更优的样本，以提高数据质量。



但其实这种bootstraping其实也在很多其他的工作中有用到了，比如llama3的训练过程，而且整个deepseek-R1的训练过程其实和llama3是非常相似的，思维链的蒸馏和google的一篇文章也非常相似《LARGE LANGUAGE MODELS CAN SELF-IMPROVE》https://arxiv.org/pdf/2210.11610

![image-20250417175212463](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417175212463.png)



### Distill

![image-20250417214923983](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417214923983.png)



## R1-follow up

### s1

s1: Simple test-time scaling：https://arxiv.org/pdf/2501.19393

这篇文章提出质量 > 数量，R1的成功因为其800K的高质量sft data，600K的reasoning data是提升R1 reasoning能力的关键，所以提出不需要600K了，只需要1K高质量data就可以。

R1的成功一部分也要来自long CoT，R1是通过RL方法使得模型自主产生更长的思维链，有没有更简单易行的方法呢？s1这篇提出将终止思考的那个token强行换成wait这个token。

![image-20250417220604279](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417220604279.png)



### LIMO

关于sft阶段的reasoning data，还有一篇文章，https://arxiv.org/pdf/2502.03387

![image-20250417221427242](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417221427242.png)

![image-20250417221714654](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417221714654.png)

![image-20250417221806596](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417221806596.png)

### Overthinking & Underthinking

Do NOT Think That Much for 2+3=? On the Overthinking of o1-Like LLMs：https://arxiv.org/pdf/2412.21187,这篇文章主要是研究overthinking的问题

Thoughts Are All Over the Place: On the Underthinking of o1-Like LLMs：https://arxiv.org/pdf/2501.18585，研究Underthinking的问题



## GRPO / DeepseekMath

DeepseekMath的训练过程也是经典的LLM训练范式

```
1. math data --> pre-train
2. sft
3. RL GRPO
```

### pretrain

先在一个1.3B的模型上训练试验：

首先构造math data:

![image-20250417223020007](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417223020007.png)

FastText Model是一个比较小的二分类模型，判断网页是否数学相关。上面的过程循环4次后结束，得到一个高质量的数据集：

![image-20250417223424694](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417223424694.png)

然后在7B的模型上训练，DeepSeekMath-Base模型来自DeepSeek-Coder，发现pretrain后，DeepSeekMath-Base很强

> Our model is initialized with DeepSeek-Coder-Base-v1.5 7B

![image-20250417224018311](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417224018311.png)

### sft

该环节需要收集一些和数学相关的instruct tuning dataset

```
1. CoT data（这部分数据不再赘述）
2. PoT data
3. tool-integrated reasoning format（简单来说是结合了CoT和PoT）
```

![image-20250417225511908](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417225511908.png)

什么是PoT（直接让LLM解决问题比较难，让LLM通过program的方法解决问题）：

![image-20250417224548012](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417224548012.png)

什么是tool-integrated reasoning format？简单来说是结合了CoT和PoT

![image-20250417225249755](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417225249755.png)

### RL

Refrence Model：做完sft后的model，即RL的起始model

![image-20250417231539390](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250417231539390.png)

在Deepseek-Math论文中分别实验了orm和prm，但是应该是发现了一些问题，所以在后续R1工作中并没有再使用reward model，而是使用rulep-based方法







