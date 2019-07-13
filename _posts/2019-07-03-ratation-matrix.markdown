---
layout: post
title: 旋转矩阵深度剖析
date: 2019-07-03 10:07:24.000000000 +09:00
img:  one-piece/one-piece7.jpg # Add image post (optional)
tag: [数学]
---

## 正交矩阵
正交矩阵的定义如下：如果

$$ AA^T = E $$

其中E为单位矩阵，则称n阶实矩阵A为正交矩阵。

所以正交矩阵的性质如下：
1. 正交矩阵的每一列、行都是单位向量，并且两两正交。最简单的正交矩阵就是单位阵。

2. 正交矩阵的逆等于正交矩阵的转置。由此可以推断出正交矩阵的行列式的值肯定为正负1。

> 所有的矩阵都可以看成一种变换。正交矩阵的变换可以看成，如果作用在一组空间基向量上，它会**将一组空间基向量(单位正交的)变换成另外一组空间基向量（还是单位正交的）**。

## 向量与坐标
对于三维空间中的任意一个向量$a$，它在一组基$\left(e_{1}, e_{2}, e_{3}\right)$下的表示如下：

$$
\boldsymbol{a}=\left[\boldsymbol{e}_{1}, \boldsymbol{e}_{2}, \boldsymbol{e}_{3}\right]\left[\begin{array}{c}{a_{1}} \\ {a_{2}} \\ {a_{3}}\end{array}\right]=a_{1} e_{1}+a_{2} \boldsymbol{e}_{2}+a_{3} \boldsymbol{e}_{3}
$$

两个向量$a,b$的内积可以写成：

$$
a \cdot b=a^{\mathrm{T}} b=\sum_{i=1}^{3} a_{i} b_{i}=|\boldsymbol{a}||\boldsymbol{b}| \cos \langle\boldsymbol{a}, \boldsymbol{b}\rangle
$$

外积写成：

$$
\boldsymbol{a} \times \boldsymbol{b}=\left\|\begin{array}{ccc}{\boldsymbol{e}_{1}} & {\boldsymbol{e}_{2}} & {\boldsymbol{e}_{3}} \\ {a_{1}} & {a_{2}} & {a_{3}} \\ {b_{1}} & {b_{2}} & {b_{3}}\end{array}\right\|=\left[\begin{array}{cc}{a_{2} b_{3}-a_{3} b_{2}} \\ {a_{3} b_{1}-a_{1} b_{3}} \\ {a_{1} b_{2}-a_{2} b_{1}}\end{array}\right]=\left[\begin{array}{ccc}{0} & {-a_{3}} & {a_{2}} \\ {a_{3}} & {0} & {-a_{1}} \\ {-a_{2}} & {a_{1}} & {0}\end{array}\right] \boldsymbol{b} \triangleq \boldsymbol{a}^{\wedge} \boldsymbol{b}
$$

其中$\boldsymbol{a}^{\wedge}$是向量$a$对应的反对称矩阵：

$$
\boldsymbol{a}^{\wedge}=\left[\begin{array}{ccc}{0} & {-a_{3}} & {a_{2}} \\ {a_{3}} & {0} & {-a_{1}} \\ {-a_{2}} & {a_{1}} & {0}\end{array}\right]
$$

## 旋转矩阵(是正交矩阵的证明)
旋转矩阵，对应了一种变换，在三维空间中，这种变换可以看成将空间中物体绕一条轴旋转一定角度。很容易发现，这是一种正交矩阵。

证明的话，可以按照如下思路。

- 首先对于一组单位正交基$\left(e_{1}, e_{2}, e_{3}\right)$经过一次旋转变换后变成了$\left(e_{1}^{\prime}, e_{2}^{\prime}, e_{3}^{\prime}\right)$ 

- 那么，对于同一个向量$a$(没有随着坐标系的旋转而发生运动)，所以会有如下等式成立，其中两个竖向量分别是$a$在两个单位正交基下的坐标：

$$
\left[e_{1}, e_{2}, e_{3}\right]\left[\begin{array}{l}{a_{1}} \\ {a_{2}} \\ {a_{3}}\end{array}\right]=\left[e_{1}^{\prime}, e_{2}^{\prime}, e_{3}^{\prime}\right]\left[\begin{array}{c}{a_{1}^{\prime}} \\ {a_{2}^{\prime}} \\ {a_{3}^{\prime}}\end{array}\right]
$$

- 上述公式左右同时左乘$\left[\begin{array}{c}{e_{1}^{\mathrm{T}}} \\ {e_{2}^{\mathrm{T}}} \\ {e_{3}^{\mathrm{T}}}\end{array}\right]$，我们会得到如下公式，其中$R$是这次旋转变换对向量$\left[a_{1}^{\prime}, a_{2}^{\prime}, a_{3}^{\prime}\right]^{\mathrm{T}}$的变换矩阵。

$$
\left[\begin{array}{c}{a_{1}} \\ {a_{2}} \\ {a_{3}}\end{array}\right]=\left[\begin{array}{ccc}{e_{1}^{\mathrm{T}} e_{1}^{\prime}} & {e_{1}^{\mathrm{T}} e_{2}^{\prime}} & {e_{1}^{\mathrm{T}} e_{3}^{\prime}} \\ {e_{2}^{\mathrm{T}} \boldsymbol{e}_{1}^{\prime}} & {\boldsymbol{e}_{2}^{\mathrm{T}} \boldsymbol{e}_{2}^{\prime}} & {\boldsymbol{e}_{2}^{\mathrm{T}} \boldsymbol{e}_{3}^{\prime}} \\ {\boldsymbol{e}_{3}^{\mathrm{T}} \boldsymbol{e}_{1}^{\prime}} & {\boldsymbol{e}_{3}^{\mathrm{T}} \boldsymbol{e}_{2}^{\prime}} & {\boldsymbol{e}_{3}^{\mathrm{T}} \boldsymbol{e}_{3}^{\prime}}\end{array}\right]\left[\begin{array}{c}{a_{1}^{\prime}} \\ {a_{2}^{\prime}} \\ {a_{3}^{\prime}}\end{array}\right] = \boldsymbol{R} a^{\prime}
$$

此处，矩阵$R$**由两组基之间的内积**组成，刻画了前后同一个向量经过一次旋转变换的坐标变换关系。因此，这个$R$就是一个旋转矩阵（只有两个正交基不变，它就不变）。这个矩阵按列看，是第二组基向量在第一组中的坐标表示，按行看，是第一组基向量在第二组中的坐标表示。它们肯定是相互正交的。

> 我们可以很清楚的发现，旋转矩阵R是一个**正交矩阵**。因为，如果将矩阵$R$转置，其实刚好就是一个$R$的逆操作，将两组正交基顺序交换了。

## 旋转向量与旋转矩阵的指数映射
指数映射的数学意义就是罗德里杰斯公式。

对于一个旋转向量a，其对应的矩阵是$R_a$，则有：

- 指数映射：$ exp(\mathbf{a}^{\wedge}) = R_a $

- 这个由叉乘时向量转矩阵形式决定，$ (\mathbf{a}^{\wedge})^T = - \mathbf{a}^{\wedge} $

- 这个右叉乘交换性质决定： $ \mathbf{a}^{\wedge} \mathbf{b} = - \mathbf{b}^{\wedge} \mathbf{a}$

- 矩阵叉乘性质：$R w^{\wedge} R ^{T} = (Rp)^{\wedge} $

- SO(3)的伴随性质： $ \mathbf{R}^{T} \exp \left(\boldsymbol{\phi}^{\wedge}\right) \mathbf{R}=\exp \left(\left(\mathbf{R}^{T} \boldsymbol{\phi}\right)^{\wedge}\right) $

- 扰动模型1

$$
\ln \left(\mathbf{R} \exp \left(\phi^{\wedge}\right)\right)^{\vee}=\ln (\mathbf{R})^{\vee}+\mathbf{J}_{r}^{-1} \boldsymbol{\phi}
$$

- BCH公式

$$
R_1 R_2 = \textbf{exp}({\phi_1}^{\wedge})\textbf{exp}({\phi_2}^{\wedge}) \approx \begin{equation} \begin{cases} \textbf{exp}(( J_l(\phi_2)^{-1} \phi_1 + \phi_2    )^{\wedge})  & \textbf{if  } \phi_1 \textbf{ is small}\\ \textbf{exp}(( J_r(\phi_1)^{-1} \phi_2+ \phi_1    )^{\wedge})   & \textbf{if  } \phi_2 \textbf{ is small} \end{cases} \end{equation}
$$


