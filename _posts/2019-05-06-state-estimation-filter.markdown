---
layout: post
title: 状态估计之卡尔曼滤波
date: 2019-05-06 10:07:24.000000000 +09:00
img:  sword/sword14.jpg # Add image post (optional)
tag: [数学]
---
本文主要介绍如何使用滤波器(卡尔曼滤波器与扩展卡尔曼滤波器)进行状态估计。

# 状态估计问题
首先列出状态估计方程，由运动方程与观测方程组成

$$
\left\{\begin{array}{l}{\boldsymbol{x}_{k}=f\left(\boldsymbol{x}_{k-1}, \boldsymbol{u}_{k}\right)+\boldsymbol{w}_{k}} \\ {\boldsymbol{z}_{k}=h\left(\boldsymbol{x}_{k}\right)+\boldsymbol{\upsilon}_{k}}\end{array}\right. \quad k=1, \ldots, N
$$

其中
- $x_{k}$   是相机在k时刻的位姿，我们一般可以使用变换矩阵或者其李代数表示它。
- $u_{k}$   是k时刻系统的输入（在slam中通常是运动传感器的读数）
- $w_{k}$   是运动过程中的噪声
- $z_{k}$   是指在$x_k$处进行了一次观测得到的观测值
- ${\upsilon}_{k}$ 是观测过程中的噪声
 
# 贝叶斯法则
对于上述状态估计问题，我们想做的是估计出现在状态的分布

$$
P\left(\boldsymbol{x}_{k} | \boldsymbol{x}_{0}, \boldsymbol{u}_{1 : k}, \boldsymbol{z}_{1 : k}\right)
$$

按照贝叶斯法则，我们可以有如下结论

$$
P\left(\boldsymbol{x}_{k} | \boldsymbol{x}_{0}, \boldsymbol{u}_{1 : k}, \boldsymbol{z}_{1 : k}\right) \propto P\left(\boldsymbol{z}_{k} | \boldsymbol{x}_{k}\right) P\left(\boldsymbol{x}_{k} | \boldsymbol{x}_{0}, \boldsymbol{u}_{1 : k}, \boldsymbol{z}_{1 : k-1}\right)
$$

其中左边是**后验**概率分布（k时刻状态已经发生），右边第一项是**似然**概率分布，第二项是**先验**概率分布（k时刻状态还没发生）。

# 卡尔曼滤波
要使用卡尔曼滤波方法对上述后验概率分布进行估计，我们首先需要两条假设
1. 上述系统满足**马尔可夫性**，当前状态只与前一状态有关
2. 系统是**线性高斯**系统，高斯分布经过线性变换仍然是高斯分布

满足上述假设的状态估计问题可以重写为

$$
\left\{\begin{array}{l}{\boldsymbol{x}_{k}=\boldsymbol{A}_{k} \boldsymbol{x}_{k-1}+\boldsymbol{u}_{k}+\boldsymbol{w}_{k}} \\ {\boldsymbol{z}_{k}=\boldsymbol{C}_{k} \boldsymbol{x}_{k}+\boldsymbol{v}_{k}}\end{array}\right. \quad k=1, \ldots, N
$$

其中第一个为运动方程，第二个为状态方程，并且**所有的状态与噪声均满足高斯分布**。这里噪声满足如下零均值高斯分布

$$
\boldsymbol{w}_{k} \sim N(\mathbf{0}, \boldsymbol{R}) . \quad \boldsymbol{v}_{k} \sim N(\mathbf{0}, \boldsymbol{Q})
$$

由于马尔科夫性，假设我们已经有了$k-1$时刻状态的**后验概率分布**：均值 $$\hat{x}_{k-1}$$ 与协方差 $$\hat{P}_{k-1}$$ ，现在需要做的是**根据k时刻的输入与观测数据，确定$x_{k}$的后验概率分布**。

> 此处需要先声明，对于噪声的协方差R与Q，为了方便我们省略了他们的下标。而我们用尖帽子表示后验，横线表示先验。

卡尔曼滤波的**第一步**是依据运动方程确定$x_{k}$的先验概率分布。如下

$$
P\left(\boldsymbol{x}_{k} | \boldsymbol{x}_{0}, \boldsymbol{u}_{1 : k}, \boldsymbol{z}_{1 : k-1}\right)=N\left(\boldsymbol{A}_{k} \hat{\boldsymbol{x}}_{k-1}+\boldsymbol{u}_{k}, \boldsymbol{A}_{k} \hat{\boldsymbol{P}}_{k-1} \boldsymbol{A}_{k}^{T}+\boldsymbol{R}\right)
$$

其中，利用线性高斯系统的性质，我们得到$x_{k}$的先验分布也是高斯分布，并且均值与方差如下：

$$
\overline{\boldsymbol{x}}_{k}=\boldsymbol{A}_{k} \hat{\boldsymbol{x}}_{k-1}+\boldsymbol{u}_{k}, \quad \overline{\boldsymbol{P}}_{k}=\boldsymbol{A}_{k} \hat{\boldsymbol{P}}_{k-1} \boldsymbol{A}_{k}^{T}+\boldsymbol{R}
$$

**下一步，我们由观测方程得到似然分布**

$$
P\left(\boldsymbol{z}_{k} | \boldsymbol{x}_{k}\right)=N\left(\boldsymbol{C}_{k} \boldsymbol{x}_{k}, \boldsymbol{Q}\right)
$$

**然后**，由贝叶斯法则我们知道，后验概率分布与似然分布和先验概率分布的乘积成正比，即

$$
N\left(\hat{\boldsymbol{x}}_{k}, \hat{\boldsymbol{P}}_{k}\right) \propto N\left(\boldsymbol{C}_{k} \boldsymbol{x}_{k}, \boldsymbol{Q}\right) \cdot N\left(\overline{\boldsymbol{x}}_{k}, \overline{\boldsymbol{P}}_{k}\right)
$$

**因为上述三个分布都是高斯分布，上述等式两侧又成正比，所以我们无需关系高斯分布系数部分内容，只需要保证等式两侧指数部分是相等的即可**。即得到下式

$$
\left(\boldsymbol{x}_{k}-\hat{\boldsymbol{x}}_{k}\right)^{T} \hat{\boldsymbol{P}}_{k}^{-1}\left(\boldsymbol{x}_{k}-\hat{\boldsymbol{x}}_{k}\right)=\left(\boldsymbol{z}_{k}-\boldsymbol{C}_{k} \boldsymbol{x}_{k}\right)^{T} \boldsymbol{Q}^{-1}\left(\boldsymbol{z}_{k}-\boldsymbol{C}_{k} \boldsymbol{x}_{k}\right)+\left(\boldsymbol{x}_{k}-\overline{\boldsymbol{x}}_{k}\right)^{T} \boldsymbol{P}_{k}^{-1}\left(\boldsymbol{x}_{k}-\overline{\boldsymbol{x}}_{k}\right)
$$

为了保证上面等式成立，我们将两边展开，另$x_{k}$的二次系数与一次系数两边相等。对于**二次系数**，我们得到

$$
\hat{\boldsymbol{P}}_{k}^{-1}=\boldsymbol{C}_{k}^{T} \boldsymbol{Q}^{-1} \boldsymbol{C}_{k}+\overline{\boldsymbol{P}}_{k}^{-1}
$$

这个等式给出了$x_{k}$分布的协方差的计算过程，此处另$$ \boldsymbol{K}=\hat{\boldsymbol{P}}_{k} \boldsymbol{C}_{k}^{T} \boldsymbol{Q}^{-1} $$，并将上式左右同时乘以$$\hat{P}_{k}$$，有

$$
\boldsymbol{I}=\hat{\boldsymbol{P}}_{k} \boldsymbol{C}_{k}^{T} \boldsymbol{Q}^{-1} \boldsymbol{C}_{k}+\hat{\boldsymbol{P}}_{k} \overline{\boldsymbol{P}}_{k}^{-1}=\boldsymbol{K} \boldsymbol{C}_{k}+\hat{\boldsymbol{P}}_{k} \overline{\boldsymbol{P}}_{k}^{-1}
$$

于是，我们得到卡尔曼滤波中$$\hat{P}_{k}$$的更新公式

$$
\hat{\boldsymbol{P}}_{k}=\left(\boldsymbol{I}-\boldsymbol{K} \boldsymbol{C}_{k}\right) \overline{\boldsymbol{P}}_{k}
$$

其中，K的更新公式是

$$
\boldsymbol{K}=\overline{\boldsymbol{P}}_{k} \boldsymbol{C}_{k}^{T}\left(\boldsymbol{C}_{k} \overline{\boldsymbol{P}}_{k} \boldsymbol{C}_{k}^{T}+\boldsymbol{Q}\right)^{-1}
$$

同样，由**一次项系数**相同可以得到

$$
-2 \hat{\boldsymbol{x}}_{k}^{T} \hat{\boldsymbol{P}}_{k}^{-1} \boldsymbol{x}_{k}=-2 \boldsymbol{z}_{k}^{T} \boldsymbol{Q}^{-1} \boldsymbol{C}_{k} \boldsymbol{x}_{k}-2 \overline{\boldsymbol{x}}_{k}^{T} \overline{\boldsymbol{P}}_{k}^{-1} \boldsymbol{x}_{k}
$$

整理可得卡尔曼滤波中$\hat{x}_{k}$的更新公式

$$
\begin{aligned} \hat{\boldsymbol{x}}_{k} &=\hat{\boldsymbol{P}}_{k} \boldsymbol{C}_{k}^{T} \boldsymbol{Q}^{-1} \boldsymbol{z}_{k}+\hat{\boldsymbol{P}}_{k} \overline{\boldsymbol{P}}_{k}^{-1} \overline{\boldsymbol{x}}_{k} \\ &=\boldsymbol{K} \boldsymbol{z}_{k}+\left(\boldsymbol{I}-\boldsymbol{K} \boldsymbol{C}_{k}\right) \overline{\boldsymbol{x}}_{k} \\ &=\overline{\boldsymbol{x}}_{k}+\boldsymbol{K}\left(\boldsymbol{z}_{k}-\boldsymbol{C}_{k} \overline{\boldsymbol{x}}_{k}\right) \end{aligned}
$$

## 卡尔曼滤波步骤归纳
由上面推导，我们可以将卡尔曼滤波归纳成如下**预测**与**更新**两个步骤：

- 预测：

$$
\overline{\boldsymbol{x}}_{k}=\boldsymbol{A}_{k} \hat{\boldsymbol{x}}_{k-1}+\boldsymbol{u}_{k}, \quad \overline{\boldsymbol{P}}_{k}=\boldsymbol{A}_{k} \hat{\boldsymbol{P}}_{k-1} \boldsymbol{A}_{k}^{T}+\boldsymbol{R}
$$

- 更新：
首先计算K（卡尔曼增益）

$$
\boldsymbol{K}=\overline{\boldsymbol{P}}_{k} C_{k}^{T}\left(\boldsymbol{C}_{k} \overline{\boldsymbol{P}}_{k} \boldsymbol{C}_{k}^{T}+\boldsymbol{Q}\right)^{-1}
$$

然后计算后验概率的分布

$$
\begin{aligned} \hat{\boldsymbol{x}}_{k} &=\overline{\boldsymbol{x}}_{k}+\boldsymbol{K}\left(\boldsymbol{z}_{k}-\boldsymbol{C}_{k} \overline{\boldsymbol{x}}_{k}\right) \\ \hat{\boldsymbol{P}}_{k} &=\left(\boldsymbol{I}-\boldsymbol{K} \boldsymbol{C}_{k}\right) \overline{\boldsymbol{P}}_{k} \end{aligned}
$$

> 卡尔曼滤波是线性系统的最优无偏估计

# 扩展卡尔曼滤波
> 上面的卡尔曼滤波是基于线性系统的，而我们实际工程遇到的问题一般都是非线性的，比如说slam中的运动方程与观测方程一般都是非线性函数。而一个高斯分布，经过非线性变换后，往往不再是高斯分布了。为了将卡尔曼滤波器的结果扩展到非线性系统中，我们通常会取一定的近似，将一个非高斯分布近似成高斯分布。

在扩展卡尔曼滤波器(Extended Kalman Filter, EKF)中，我们一般在某个点附近考虑运动方程及观测方程的**一阶泰勒展开**，只保留一阶项，即线性的部分，然后按照线性系统的方法进行推导.

经过线性化（一阶泰勒展开）之后，我们得到的**运动方程**如下

$$
\boldsymbol{x}_{k} \approx f\left(\hat{\boldsymbol{x}}_{k-1}, \boldsymbol{u}_{k}\right)+\left.\frac{\partial f}{\partial \boldsymbol{x}_{k-1}}\right|_{\hat{\boldsymbol{x}}_{k-1}}\left(\boldsymbol{x}_{k-1}-\hat{\boldsymbol{x}}_{k-1}\right)+\boldsymbol{w}_{k}
$$

**观测方程**如下

$$
z_{k} \approx h\left(\overline{x}_{k}\right)+\left.\frac{\partial h}{\partial x_{k}}\right|_{\overline{x}_{k}}\left(x_{k}-\hat{x}_{k}\right)+n_{k}
$$

此处，我们另两个方程中的**两个偏导数**为

$$
\boldsymbol{F}=\left.\frac{\partial f}{\partial \boldsymbol{x}_{k-1}}\right|_{\hat{\boldsymbol{x}}_{k-1}}
$$

$$
\boldsymbol{H}=\left.\frac{\partial h}{\partial \boldsymbol{x}_{k}}\right|_{\overline{\boldsymbol{x}}_{k}}
$$

然后，根据运动方程，我们得到的**先验概率分布**为

$$
P\left(\boldsymbol{x}_{k} | \boldsymbol{x}_{0}, \boldsymbol{u}_{1 : k}, \boldsymbol{z}_{0 : k-1}\right)=N\left(f\left(\hat{\boldsymbol{x}}_{k-1}, \boldsymbol{u}_{k}\right), \boldsymbol{F} \hat{\boldsymbol{P}}_{k-1} \boldsymbol{F}^{\mathrm{T}}+\boldsymbol{R}_{k}\right)
$$

根据观测方程，我们得到的**似然概率分布**为

$$
P\left(\boldsymbol{z}_{k} | \boldsymbol{x}_{k}\right)=N\left(h\left(\overline{\boldsymbol{x}}_{k}\right)+\boldsymbol{H}\left(\boldsymbol{x}_{k}-\overline{\boldsymbol{x}}_{k}\right), \boldsymbol{Q}_{k}\right)
$$

下面的推导与卡尔曼滤波相同，通过贝叶斯公式使得高斯分布中的指数部分相同。

## 扩展卡尔曼滤波步骤总结
- 预测

$$
\overline{\boldsymbol{x}}_{k}=f\left(\hat{\boldsymbol{x}}_{k-1}, \boldsymbol{u}_{k}\right), \quad \overline{\boldsymbol{P}}_{k}=\boldsymbol{F} \hat{\boldsymbol{P}}_{k} \boldsymbol{F}^{T}+\boldsymbol{R}_{k}
$$

- 更新，先计算K（卡尔曼增益）

$$
\boldsymbol{K}_{k}=\overline{\boldsymbol{P}}_{k} \boldsymbol{H}^{\mathrm{T}}\left(\boldsymbol{H} \overline{\boldsymbol{P}}_{k} \boldsymbol{H}^{\mathrm{T}}+\boldsymbol{Q}_{k}\right)^{-1}
$$

然后更新后验分布的均值与方差

$$
\hat{\boldsymbol{x}}_{k}=\overline{\boldsymbol{x}}_{k}+\boldsymbol{K}_{k}\left(\boldsymbol{z}_{k}-h\left(\overline{\boldsymbol{x}}_{k}\right)\right) \\ \hat{\boldsymbol{P}}_{k}=\left(\boldsymbol{I}-\boldsymbol{K}_{k} \boldsymbol{H}\right) \overline{\boldsymbol{P}}_{k}
$$

## EKF的评价
- 优点
    - 推导简单清楚，适用各种传感器形式
    - 易于做多传感器融合

- 缺点
    - 一阶马尔可夫性过于简单，不能充分利用所有的信息
    - 一阶泰勒展开线性化遇到非线性严重的模型会误差比较大，并且只能局部近似
    - 需要存储所有状态量的均值与协方差矩阵，存储量呈平方增长
    - 对outlier很敏感
    - 假设都是高斯分布，不是对所有情况都适用

> 还有一些其他的滤波方法，比如粒子滤波、IF（信息滤波器）、UKF（Unscented KF）等等。其中粒子滤波不是假设高斯分布。

# 参考文献
- [1]. 视觉slam十四讲 
