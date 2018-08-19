---
layout: post
title: 关系抽取论文解读End-to-End Relation Extraction using LSTMs on Sequences and Tree Structures
tags: relation-extraction
---
关系抽取是从一个句子中判断出这个句子里面两个实体之间的关系。
>比如给定句子：

```python
1	"The system as described above has its greatest application in an arrayed <e1>configuration</e1> of antenna <e2>elements</e2>."
Component-Whole(e2,e1)
Comment: Not a collection: there is structure here, organisation.

```

上面的句子是数据集[SemEval 2010 Task 8 数据集
](http://kozareva.com/downloads.html)中的一个训练集的实际样本，1表示第一条句子，`<e1>configuration</e1>` 是指明了实体一， `<e2>elements</e2>`是指明了实体二，`Component-Whole(e2,e1)`表明了两个实体之间的关系是`Component-Whole`关系。`Comment`是对句子的一些描述信息。


数据集SemEval 2010 Task 8中关系是9种，为了区别正反（`Cause-Effect(e1,e2)`与`Cause-Effect(e2,e1)`看作是两种关系）和其他（`other`,有些句子中的实体关系不是给定的关系）一共是19种关系：

```python
(1) Cause-Effect
(2) Instrument-Agency
(3) Product-Producer
(4) Content-Container
(5) Entity-Origin
(6) Entity-Destination
(7) Component-Whole
(8) Member-Collection
(9) Message-Topic
```

那么我们要根据训练集训练出的模型对句子中的关系进行预测。

```python
A few days before the service, Tom Burris had thrown into Karen's <e1>casket</e1> his wedding <e2>ring</e2>.
```
上面就是测试例子，请给出两个实体之间的关系。这就是我们要做的关系抽取，现在的关系抽取都是有监督的关系抽取，还有半监督的，还有无监督的（开放域的实体关系抽取），这在深度学习中还是研究热点。


接下来就进入正文：
#### End-to-End Relation Extraction using LSTMs on Sequences and Tree Structures
【Proceedings of the 54th Annual Meeting of the Association for Computational Linguistics, pages 1105–1116,
Berlin, Germany, August 7-12, 2016.c  2016 Association for Computational Linguistics
】

#### 论文成果：
- 1.实现端到端的实体关系抽取
- 2.实验表明词序和依存树的结合有利于关系抽取
- 3.使用Bi Tree LSTM构建网络架构

论文提出了一种端到端的神经网络模型，用于抽取实体和实体间的关系。该方法同时考虑了句子的词序信息和依存句法树的子结构信息，这两部分信息都是利用双向序列LSTM-RNN建模，并将两个模型堆叠起来，使得关系抽取的过程中可以利用实体相关的信息。

#### 论文架构

![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm1.png)

为了能够清晰的理解总体架构，我们需要拆分上述结构，理解基本的`RNN`

##### RNN结构：

![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm2.png)

![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm3.png)

![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm4.png)

RNN与DNN的区别就是输入来自两个方面，一个是正常的输入x，一个是上一次隐藏单元的值。

##### LSTM结构：

![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm5.png)

##### Bi LSTM 结构

![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm6.png)


##### Bi RNN 结构


![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm7.png)

双向RNN的前向与后向在本质上没有区别，只是数据颠倒了一下，最终的输出由前向和后向组合。

##### Tree LSTM 结构

![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm8.png)

树型LSTM的产生是由于依存分析树本身就是一颗树的形式，如果转化为简单的序列结构，可能会丢失信息。

![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm9.png)

树型LSTM与序列LSTM的区别在于有多个来源的输入，一个正常输入x，第一个隐藏单元值h1，第二个隐藏单元值h2……


#### 实验结果：
![image](http://upyun.midnight2104.com/blog/2018-8-19/bitreelstm10.png)
【SemEval 2010 Task 8  数据集】








