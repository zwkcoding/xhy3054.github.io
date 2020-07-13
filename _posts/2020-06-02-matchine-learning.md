---
layout: post
title: 机器学习中的常见函数总结（备忘）
date: 2020-06-02 10:07:24.000000000 +09:00
img:  one-piece/one-piece23.jpg # Add image post (optional)
tag: [机器学习]
---

主要近来又发现本人的记忆力水平需要笔记辅助，因此写来备忘。

# 目录

- [Softmax函数](#Softmax函数)
- [Triplet loss](#Triplet_loss)

## Softmax函数
Softmax函数，也叫归一化指数函数，主要是用来辅助多分类任务。

应用场景： 对于一个k分类的问题，通过对一个物体的特征进行处理，得到一个k维向量v，最好的结果当然是正确分类的维度取值1，其他维度取值0. 但是一般情况下我们得到的结果是每个维度取值范围为从负无穷到正无穷。

Softmax的作用：将每个维度的取值限定到0到1之间，并且所有维度的值之和为1.

Softmax的做法：
1. 首先将每个维度的值转换为非负数，做法：`v[i] = exp(v[i])`

2. 将非负数的值限定到0和1之间，并似的所有值和为1.做法：`v[i] = v[i]/(v[1]+v[2]+...+v[k])`;

总结：`v[i] = exp(v[i]) / {exp(v[1]) + exp(v[2]) + ... + exp(v[k])}`


## Triplet_loss

> Triplet loss最初是在FaceNet的论文中提出，可以较好地学到人脸的embedding，相似的图像在embedding空间里是相近的，可以判断是否是同一个人脸。

### 目的

- 使具有相同标签的样本在embedding空间尽量接近。

- 使具有不同标签的样本在embedding空间尽量远离。

如果只遵循以上两点，最后embedding空间中相同类别的样本间距离很小，不同类别的样本之间距离也会偏小。因此，需要加入margin。

### 原理

输入是一个三元组`<a, p, n>`，其中

- a：anchor 原点
- p：positive 与a同一类别的样本
- n：negative 与a不同类别的样本

最终的loss为

$$ L = max(d(a,p) - d(a,n) + margin, 0) $$

最小化L，则`d(a,p)->0，d(a,n)->margin`
