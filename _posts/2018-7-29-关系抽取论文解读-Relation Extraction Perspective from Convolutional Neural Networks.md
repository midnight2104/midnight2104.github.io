---
layout: post
title: 关系抽取论文解读-Relation Extraction Perspective from Convolutional Neural Networks
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
#### Relation Extraction:Perspective from Convolutional Neural Networks

> 【Nguyen T H, Grishman R. Relation Extraction: Perspective from Convolutional Neural Networks[C]// The Workshop on Vector Space Modeling for Natural Language Processing. 2015:39-48.
】


文章中主要使用的方法是多核卷积神经网络来进行关系抽取。
##### 论文成果：
1. 不使用复杂的NLP进行关系抽取
2. 使用位置特征来表示距离
3. 使用多核卷积构建网络架构
4. 在SemEval-2010 Task 8数据集进行了实验，取得了很好的结果

##### 论文架构：

![image](http://upyun.midnight2104.com/blog/2018-8-5/multiCNNre1.png)



##### 神经网络输入：
结合句子特征与位置特征构成卷积神经网络的最终输入

![image](http://upyun.midnight2104.com/blog/2018-8-5/multiCNNre2.png)

##### 神经网络前向传播：

![image](http://upyun.midnight2104.com/blog/2018-8-5/multiCNNre3.png)

上述过程执行4次，filter_size依次为(2 , 500) , (3 , 500) ,  (4 , 500) ,  (5 , 500),
得到结果是(512 , 1),然后进行全连接。

![image](http://upyun.midnight2104.com/blog/2018-8-5/multiCNNre4.png)


最终的实验结果F1值达到82.8
