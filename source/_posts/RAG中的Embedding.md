---
title: RAG中的Embedding
date: 2026-02-27 23:18:55
tags:
  - LLM
---


>在学习RAG时，注意到其中的Embedding机制，跟深度学习中的Embedding进行了对比，发现两者虽然原理一致，但并不能完全算是同一个东西。因此做此记录。


在构建 RAG（Retrieval-Augmented Generation）系统时， Embedding是其中最为基础的一步，RAG 的第一步不是“生成”，而是“检索”。  而检索的基础，是一个结构良好的语义向量空间。

本文将系统讲清楚：

1. RAG 中的 embedding 到底是什么
    
2. 它和传统 DNN / TextCNN 中的 embedding 是否相同
    
3. 分类损失 vs 对比损失的本质区别
    
4. 语义 embedding 是如何训练出来的
    
5. 为什么 RAG 必须依赖专门训练的 embedding
    

---

# 一、RAG 中的 Embedding 在做什么？


RAG 的流程是：

1. 文档切分成 chunk
    
2. 每个 chunk 转换为 embedding(文本 → embedding 模型 → 向量（如 1536 维）)
    
3. 存入向量库
    
4. 用户提问 → 转换为 embedding
    
5. 计算相似度 → 取 Top-k
    
6. 拼接上下文 → 交给 LLM 生成
    

可以看到，Embedding 决定了“能不能召回正确内容”。

它本质上是在构建一个语义几何空间

---

# 二、Embedding 的抽象本质

从数学角度看，所有 embedding 都是：

$f(x) \rightarrow \mathbb{R}^n$

即：把离散输入映射到高维连续向量空间。

因此：

- CNN 最后一层特征向量是 embedding
    
- DNN 隐藏层输出是 embedding
    
- TextCNN 的词向量层是 embedding
    
- RAG 的语义向量也是 embedding
    

从“神经网络结构”层面，它们没有本质区别。

真正的区别，在于损失函数优化的目标。

---

# 三、TextCNN 中的 Embedding 和 RAG 一样吗？

TextCNN 结构通常是：

token id → embedding layer → 卷积 → pooling → 全连接 → 分类

embedding layer 本质上是一个可训练矩阵：

$E \in \mathbb{R}^{|V| \times d}$

它只是把 token id 转换成向量。

从形式上看，确实和 RAG 中的 embedding 很像。

但关键问题是：

> 它是如何被训练的？

在 TextCNN 中，embedding 的参数更新来源于：

$$  
L = CrossEntropy(y_{pred}, y_{true})  
$$

也就是说：

embedding 是为了“分类准确”而训练的。

它不关心：

- 向量之间的几何距离是否合理
    
- 语义相似的文本是否靠近
    
- 空间结构是否适合检索
    

因此：

TextCNN embedding 是“任务导向表示”。

而 RAG embedding 是“空间结构导向表示”。

这才是根本差异。

---

# 四、分类损失：它在优化什么？

在传统分类模型中：

输入 → embedding → 分类头 → softmax → 标签

损失函数是交叉熵：

$$  
L = -\sum y \log(\hat{y})  
$$

它优化的是**决策边界**

模型只需要把不同类别分开。

但它不会显式优化：

- 向量之间的距离
    
- 全局语义结构
    
- 相似度排序能力
    

因此：

分类模型的 embedding 通常不能直接用于语义检索。

---

# 五、语义 Embedding 的训练方式

语义 embedding 的核心目标不是“预测类别”，而是：

> 构建一个有良好几何结构的向量空间。

这个空间必须满足：

- 语义相似 → 距离接近
    
- 语义无关 → 距离远离
    
- 排序稳定
    
- 可泛化
    

实现这个目标的核心技术是：对比学习（Contrastive Learning）



## 对比学习的基本思想

传统分类是：

$$  
x \rightarrow label  
$$

而对比学习是：

$$  
(x_i, x_j) \rightarrow similarity  
$$

模型不再预测类别，而是学习两个样本之间的关系。

这意味着训练目标从“点分类”变成了“点与点之间的几何约束”。



## Contrastive Loss（早期形式）

经典形式：

$$  
L = y \cdot d^2 + (1-y) \cdot \max(0, margin - d)^2  
$$

其中：

- $y = 1$ 表示正样本对（相似）
    
- $y = 0$ 表示负样本对（不相似）
    
- $d = | v_i - v_j |$
    
- $margin$ 是一个安全距离
    



### 情况 1：正样本对（$y=1$）

$$  
L = d^2  
$$

目标是让：

$$  
d \rightarrow 0  
$$

也就是说相似文本向量尽可能重合。



### 情况 2：负样本对（$y=0$）

$$  
L = \max(0, margin - d)^2  
$$

只有当：

$$  
d < margin  
$$

才会产生梯度。

这意味着：

- 如果已经足够远 → 不再优化
    
- 如果太近 → 强制拉开
    

这会在空间中形成一个“安全半径”。




Contrastive Loss 会形成：

- 正样本聚成团
    
- 不同语义团块之间保持至少 margin 的间隔
    

但问题是：

- 负样本利用率低
    
- 训练效率不高
    
- 空间结构不够平滑
    

因此后来出现了更先进的方法。



## InfoNCE（现代主流方法）

现代 embedding 模型（包括大型 API 模型）基本都基于 InfoNCE 或其变体



###  批量构造方式

在一个 batch 中：

$$  
(A_1, B_1), (A_2, B_2), \dots, (A_n, B_n)  
$$

其中：

- $(A_i, B_i)$ 是正样本对
    
- $B_j (j \neq i)$ 自动成为负样本
    

这叫做：

> in-batch negative sampling



###  相似度定义

通常使用余弦相似度：

$$  
sim(A_i, B_j) = \frac{v_i \cdot v_j}{|v_i||v_j|}  
$$

这意味着：

模型优化的是方向，而不是长度。



###  损失函数

$$  
L_i = -\log  
\frac{\exp(sim(A_i,B_i)/\tau)}  
{\sum_{j=1}^{N} \exp(sim(A_i,B_j)/\tau)}  
$$

这里：

- 分子：正确匹配
    
- 分母：所有可能匹配
    
- $\tau$：温度参数
    



##  InfoNCE 本质上是什么？

观察这个公式：

$$  
\frac{\exp(sim(A_i,B_i)/\tau)}  
{\sum_{j} \exp(sim(A_i,B_j)/\tau)}  
$$

它其实是：**softmax 分类**

也就是说：

模型在做一件事：

在所有$B_j$ 中，找出哪个最匹配 $A_i$
 

这等价于：**相似度分类问题。**



## 梯度直觉分析

对正样本：

- 增大 $sim(A_i,B_i)$
    
- 提高正确匹配概率
    

对负样本：

- 降低 $sim(A_i,B_j)$
    
- 让错误匹配概率下降
    

这会产生两个效果：

1. 正样本持续被拉近
    
2. 所有负样本都会被推远
    

和传统 Contrastive Loss 不同的是，每一个 batch 都会产生大量负样本梯度。

因此空间更平滑、更稳定。



## 温度参数 $\tau$ 的作用

$$  
softmax(sim / \tau)  
$$

- $\tau$ 小 → 分布更尖锐 → 强调 hardest negative
    
- $\tau$ 大 → 分布更平滑 → 训练更稳定
    

直觉上：

$$  
\tau \downarrow \Rightarrow 梯度集中在最难负样本  
$$

这会强化语义区分能力。



## 几何空间的形成过程

经过大量训练后：

- 相似文本形成高密度语义团块
    
- 不同主题形成方向分离
    
- 空间变成近似球面分布
    
- 余弦距离变成稳定排序依据
    

这就是为什么：

> embedding 可以直接用于向量检索。



## Contrastive Loss vs InfoNCE 本质区别

|维度|Contrastive|InfoNCE|
|---|---|---|
|负样本利用|单个|批量|
|空间平滑性|较弱|强|
|是否类似分类|否|是|
|收敛速度|较慢|较快|
|工程主流|早期|现代主流|


## 关键理解

分类模型优化的是：

$$  
f(x) \rightarrow label  
$$

对比学习优化的是：

$$  
f(x_i) \cdot f(x_j)  
$$

也就是说：

分类优化“点到类别”的映射。

对比学习优化“点到点”的几何关系。

这就是：

> RAG embedding 能够做检索，而普通分类 embedding 不能的根本原因。


---


# 六、空间几何结构的差异

## 分类损失 → 决策边界空间

- 类别分成若干区域
    
- 类别内部结构无序
    
- 距离不具有稳定语义意义
    

## 对比损失 → 语义几何空间

- 相似概念自然聚类
    
- 形成“语义方向”
    
- 全局结构一致
    
- 余弦相似度可直接用于排序
    

RAG 需要的是后者。

---

# 七、为什么 RAG 不能用普通分类 embedding？

假设一个模型是做情感分类训练的。

它可能学到：

- “喜欢”靠近“开心”
    
- “讨厌”靠近“生气”
    

但不会保证：

- “账号注销”接近“删除账户”
    
- “数据库索引”接近“查询优化”
    

因为这些不影响分类结果。

RAG 需要的是：

> 泛化语义匹配能力

而不是任务特定表示。

---

# 八、语义 Embedding 的训练数据来源

大型模型通常使用：

- 问答对
    
- 搜索 query-文档对
    
- 改写对
    
- 多语言对齐数据
    
- Hard negative mining
    

目标是构建一个：

> 稳定、泛化、可检索的语义空间。

---

# 九、最终总结

从神经网络机制看：

所有 embedding 都是：

$f(x) \rightarrow \mathbb{R}^n$

但决定 embedding 性质的不是模型结构，而是：

> 损失函数在优化什么。

可以用一句话总结：

- 分类 embedding 优化决策边界
    
- 语义 embedding 优化空间几何结构
    

RAG 之所以依赖 embedding，  
是因为它的本质问题是：

> 信息检索问题

而信息检索需要一个结构良好的语义空间。
