
# ICD-9-CM-3 介绍

- ICD-9-CM的全称为**国际疾病分类临床修改版，第九次修订版**
- 其中ICD-9-CM的第1卷和第2卷用于诊断代码
- 第3卷，即ICD-9-CM-3，用于对医疗程序进行分类以用于计费目的的程序代码系统

------

## 主要目标

- **在不依靠详细手术编码，即表中deail_code，的情况下对ICD9-CM3的手术名称进行聚类**
- 聚类后可以对绝大多数的类别进行描述，描述的内容对该类下的所有内容具有唯一性

------

## 实现方法及步骤

1. 数据清理
2. **分词**
3. **词向量模型训练**
4. 短文本（手术名称）向量计算
5. **文本聚类**
6. 聚类效果评估

------

# 基础数据

<img src="C:\Users\MYTh_\Desktop\数据预览.PNG" style="zoom:100%">

------

## 数据清理

- 去除“detail_name” 中不需要或是无意义的词
  - **(null)** 表示该手术没有详细代码
  - **[ ] 方括号**中的内容为同义词、替换词或解释语
  - **( )圆括号**用以表示补充词
  - 部分严重影响聚类结果且无意义的高频词，如“术”，“性”等
- 分词
  - [jieba](https://github.com/Embedding/Chinese-Word-Vectors/tree/master/testsets)
  - ~~CoreNLP~~
  - ~~SnowNLP~~

------

## 分词前后对比

![](C:\Users\MYTh_\Desktop\token.PNG)

------

## 高频词(出现频率>30)分布

<img src="C:\Users\MYTh_\Desktop\下载.png" style="zoom:55%">

------

## 词向量训练

- 为什么使用`Word2Vec`？
  - **`Word2vec` 本质上是一种把词语从 `one-hot encoder` 的形式，表示成向量形式的降维操作**，通过最终得到的向量完成聚类
  - 需要区分手术名称中的语义关系
- 不使用已经[训练好的词向量](https://github.com/Embedding/Chinese-Word-Vectors)的原因是：
  - 主要训练对象是维基百科和报纸上的文本，所以对医学类词语的训练结果不准确

------

## 训练过程

```python
model = Word2Vec(sentences=LineSentence('~/segment.txt'),
                 size = 500, 
                 min_count = 1,
                 sg=1) 
```

- 生成的词向量从10维到1000维（间隔50维）
- 提取同一个关键词，如“治疗”，比较起相似度最大的十个词语
- 当训练维度 > 500之后，最相似的10个词语基本开始停止变化，于是取500维作为最终训练维度

------

## 短文本向量计算

- 常规计算文本向量的方法有
  - ==均值Word2Vec==
  - ~~Sent2Vec~~
    - 数据集较小，不适用
  - ~~TF-IDF~~
    - 运行效率异常缓慢，且结果没有显著提高

------

## 文本聚类

- 使用`K-means`实现文本聚类

```python
kmeans = cluster.KMeans(n_clusters = 5000,max_iter = 2000)
kmeans.fit(vector_list)
```

- 生成标签并标记原数据

```python
labels= kmeans.predict(vector_list)
icd9v3["label"] = labels
icd9v3_sorted = icd9v3.sort_values(by = ['label'], 
				   ascending = (True))
```

------

## 目前的困难—高频词取舍

- 分词后会出现部分无法被舍弃的高频次，如“切除，切开，其他”，这样的词语会对分类结果产生负面影响，但往往无法知道这样的词是否可以被舍弃
  ![](C:\Users\MYTh_\Desktop\df1.PNG)

------

## 目前的困难—分词

1. 无法判断分类后的合理性与准确性
   ![](C:\Users\MYTh_\Desktop\df2.PNG)
2. 因为词频太低导致的分类错误
   ![](C:\Users\MYTh_\Desktop\df4.PNG)
3. 关键词异常
   ![](C:\Users\MYTh_\Desktop\df3.PNG)

------

## 目前的困难—其他

- 词嵌入只是目前可以被想到处理这个问题最好的方法，但对与未知的算法内不知道是否有更好的算法可以替代
- 分类后依然有5000个类别，相比原先的10316之缩小的一倍

------

## 未来优化方案

1. 如果每个手术都能有详细的介绍，那学习的文本数量能大幅增加，从而获得更可靠的词向量
2. 医学方面的停词表的建立
3. 加入其他参数，如：手术费用，每年实施这个手术的人数等
