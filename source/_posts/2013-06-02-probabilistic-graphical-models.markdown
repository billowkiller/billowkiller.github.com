---
layout: post
title: "Probabilistic Graphical Models"
date: 2013-06-02 13:49
comments: true
categories: 
tags: 
---

### 1. PGM介绍

GM(graphical model)，图模型首先它是一个概率模型，用的是图的数据结构，用来表示随机变量之间的以来关系。
因为是涉及到变量之间的关系，于是在概率理论特别是统计部分得到很长远的应用，另外还包括机器学习。

可以看的出来，概率图模型是综合了统计和计算机技术：

*   统计方面提供了概率基础，包括概率分布的分析，不确定性的处理，决策判定等等。
*   计算机方面可以将概率中的条件概率置于高维空间中，用图来表示它们的数据结构，并使用高效的算法计算。

使用现实中得到的知识数据来表述图模型，作为declarative representation，它与我们所要使用的算法是
松耦合的，可以将不同的算法应用到图模型上，在图模型中可以加入专家知识，和从其他数据中学习到的知识。

**什么时候需要用到PGM？**

*   拥有噪音数据或者不确定性比较大的时候
*   拥有很多先验知识的时候, 要比普通的机器学习来的有效
*   多个变量的推理(reason)，特别是多个变量之间存在内在联系
*   从较小的模块化的模型中建立复杂的模型，疾病诊断或错误诊断

**应用**

概率图模型能够发现和分析复杂分布的结构，提取出原本非结构化的数据，使得这些分布能够被有效的重构和利用。
主要在图像和视频智能信息处理领域已有应用。具体的应该用包括：

*   Medical diagnosis
*   Fault diagnosis
*   Natural language processing
*   speech recognition
*   Social network models
*   computer vision
    *   Image segmentation
    *   3D reconstruction 
    *   Holistic scene analysis
*   decoding of low-density parity-check codes
*   Robot localization & mapping

概率图理论共分为三个部分，分别为**表示理论，推理理论和学习理论**。

### 2. 两种模型

先说说**生成模型和判别模型**。

假设有观察值序列$Y\=(Y\_1,Y\_2,...,Y\_n)$，求其对应的状态序列$X\=(X\_1,X\_2,...,X\_n)$，则实际上即使求出状态序列$X^\*$，使得条件概率$p(X\_1,X\_2,...,X\_n\|Y\_1,Y\_2,...,Y\_n)$最大化，即\:

$$
\begin{align}
X^* = arg max_{X_1,X_2,...,X_n}p(X_1,X_2,...,X_n|Y_1,Y_2,...,Y_n)
\end{align}
$$

#### 2.1 生成模型

生成模型\(Generative Models\)不直接对$p(X\_1,X\_2,...,X\_n\|Y\_1,Y\_2,...,Y\_n)$进行建模，而是先对其进行变换，构建联合概率$p(生成模型和判别模型。X\_1,X\_2,...,X\_n\,Y\_1,Y\_2,...,Y\_n)$，即

$$
\begin{equation}
\begin{split}
p(X_1,X_2,...,X_n|Y_1,Y_2,...,Y_n) &= \frac{p(X_1,X_2,...,X_n,Y_1,Y_2,...,Y_n)}{p(Y_1,Y_2,...,Y_n)} \\
&= \frac{p(Y_1,Y_2,...,Y_n|X_1,X_2,...,X_n) × p(X_1,X_2,...,X_n)}{p(Y_1,Y_2,...,Y_n)}
\end{split}
\end{equation}
$$

在给定观察值序列的前提下，其出现的概率是一定的，所以得到

$$
\begin{equation}
\begin{split}
X^* &= arg max_{X_1,X_2,...,X_n}p(X_1,X_2,...,X_n|Y_1,Y_2,...,Y_n) \\
&= arg max_{X_1,X_2,...,X_n}p(Y_1,Y_2,...,Y_n|X_1,X_2,...,X_n) × p(X_1,X_2,...,X_n)
\end{split}
\end{equation}
$$

生成模型认为观测值是由状态生成的。

#### 2.2 判别模型

判别模型\(Discriminative Models\)克服了生成模型的独立性假设，其直接对条件概率$p(X\_1,X\_2,...,X\_n\|Y\_1,Y\_2,...,Y\_n)$进行建模，也就是说，在给定观察序列的条件下，寻找最可能的状态序列的时候，条件分布可以直接使用。

#### 2.3 生成模型和判别模型对比

*	统计建模方式不同。生成模型构建联合分布$p(X,Y)$；判别模型构建条件概率分布$p(X\|Y)$。
*	训练时，二者优化准则不同。生成模型优化训练数据的联合分布概率；判别模型优化训练数据的条件分布概率，判别模型与序列标记问题有较好的对应性。
*	训练复杂度不同。判别模型训练复杂度较高。
*	对于观察序列的处理不同。生成模型中，观察序列作为模型的一部分；判别模型中，观察序列只作为条件，因此可以针对观察序列设计灵活的特征。
*	是否支持无监督训练。生成模型支持无监督训练。
*	应用角度不同。生成模型一般主要是对后验概率建模，从统计的角度表示数据的分布情况，能够反映同类数据本身的相似度；判别模型主要特点是寻找不同类别之间的最优分类面，反映的是异类数据之间的差异。

#### 2.4 举例

常见的Generative Model主要有：

*	Gaussians, Naive Bayes, Mixtures of multinomials
*	Mixtures of Gaussians, Mixtures of experts, HMMs
*	Sigmoidal belief networks, Bayesian networks
*	Markov random fields

常见的Discriminative Model主要有：

*	logistic regression
*	SVMs
*	traditional neural networks
*	Nearest neighbor

### 3. 表示理论

不同的概率图的模型可以分成3类：有向图模型，无向图模型和混合概率图模型。

#### 3.1 有向图模型

有向图概率模型使用有向边连接不同的结点，这些有向边通常表示了结点间的因果关系。典型的代表是隐马尔可夫模型，贝叶斯网络和的动态贝叶斯网络。

* HMM(Hidden Markov Model)

  看这个名称就知道，隐马尔可夫模型是马尔可夫模型的扩展，它是在马尔可夫链的基础上发展起来的。马尔可夫链是马尔可夫随机过程的特殊情况，即马尔可夫链的状态和时间参数都是离散的马尔可夫过程。它的$m+1$时刻的状态只是和$m$时刻的状态有关，而与之前的状态无关。实际中，马尔可夫链的每一状态可以对应于一个可观测到的物理事件。比如天气预测中的雨晴雪等，根据这个模型可以算出各种天气在某一时刻出现的概率。
  
  由于实际问题比马尔可夫链模型所描述的更为复杂，观察到的事件并不是与状态一一对应的，而是通过一组概率分布相联系，这个就是HMM。HMM是一个双重的随机过程，其中之一是马尔可夫链，描述状态转移。另外一个随机过程描述状态和观察值，它并不是与状态一一对应的，需要通过一个随机过程去感知状态的存在及特性。这个就是Hidden的由来。

  <img src="http://upload.wikimedia.org/wikipedia/commons/thumb/8/83/Hmm_temporal_bayesian_net.svg/500px-Hmm_temporal_bayesian_net.svg.png" width="400px" alt="一个典型的HMM，y为观察值，x为状态值，t为时刻"/>

  具体的一个例子为phone HMM，这是一个语音识别的例子，根据单词的发言可以建立HMM，识别出得到的声音模型为观察值evidents，从start到end计算得到概率最大的HMM最终状态即为识别到的单词。
  
  <img src="http://i1113.photobucket.com/albums/k512/billowkiller/LinkSource/hmm_zpsf0b66b9e.png" width="400px" alt="一个典型的HMM，y为观察值，x为状态值，t为时刻"/>
  
  * 贝叶斯网络(Beyesian Network)
  
  一般是指带有概率信息的有向无环图(directed acyclic graph, DAG)。由两部分组成：
  
  * 图中的节点表示的是随机变量，结点间的连接表示了可能的因果关系。
  * 每一个结点都附有与该变量相联系的条件概率分布函数(Conditional Probability Distribution, CPD)，如果变量是离散的，则表现为条件概率表(Conditional Probability Table, CPT)。
  
  贝叶斯网络表现的是一个联合概率分布，可以通过Chain Rule来计算$p(X\_1,X\_2,...,X\_n)=\prod{p(X\_i\|Par_G(X\_i))}$。
  
#### 3.2 无向图模型

无向图概率模型用来建立随机变量间的空间相互关系或只是相互依赖性。典型的代表是马尔科夫随机场和条件随机场。