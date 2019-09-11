---
layout: post
title: vins中基于滑窗的后端优化
date: 2019-09-03 10:07:24.000000000 +09:00
img:  one-piece/one-piece15.jpg # Add image post (optional)
tag: [多传感器融合]
---

前面已经讲了很多vins里的细节操作，今天来谈谈将整个系统最核心的后端优化部分。

# vins的后端优化
一般在vins系统中，视觉与imu数据经过预处理后，此时如果已经完成初始化，则会在滑窗中进行非线性优化操作。整个过程一般会是如下过程：

1. 对滑窗中所有特征点进行三角化操作

2. 针对现有的窗口中的数据，构建非线性优化问题，并为该问题添加优化参数与待优化残差项，然后进行优化；

3. 优化完成后进行滑窗操作（根据之前判断的次新帧是否作为关键帧保留，而选择是marg掉最老的关键帧或者marg掉次新帧）；

> 其中2,3两步是整个滑窗优化问题的核心操作。

## 参与优化的参数
在vins的后端优化中，**优化的参数变量**很多，主要有

1. 每一帧body系的位姿；

2. 每一帧body系的速度；

3. 每一帧时imu的bias;

4. 每个特征点在可以观测到它的第一帧相机坐标系下的逆深度；

5. 相机与body系的外参；

6. 相机与imu测量的时间偏差（如果存在）;

> 其中imu测量的预积分残差会对1,2,3进行优化，视觉重投影残差会对1,4,5,6进行优化。

## 待优化的残差项
在vins的后端优化中，**待优化的残差项**也有一些，主要有
1. imu测量的预积分残差；

2. 视觉观测的重投影误差；

3. 先验残差；

4. 闭环检测残差；

> 在下面讲到优化过程时，涉及到很多求导雅克比矩阵的操作，这些求导操作是有明显的流程的。首先需要明确被求导的因变量，与求导的自变量。然后如果是关于姿态（四元数）的求导，就先将旋转变换到李群上，使用扰动模型去推导。其他的比较简单直接推就行了。

## imu测量的预积分残差
在滑窗中，首先被添加到问题中的残差项就是imu的残差。imu残差其实就是imu预积分的误差，计算方式如下（此处计算的残差是i帧与j帧之间的预积分残差）：

$$
\left[\begin{array}{c}{\mathbf{r}_{p}} \\ {\mathbf{r}_{q}} \\ {\mathbf{r}_{v}} \\ {\mathbf{r}_{b a}} \\ {\mathbf{r}_{b g}}\end{array}\right]_{15 \times 1}=\left[\begin{array}{c}{\mathbf{q}_{b_{i} w}\left(\mathbf{p}_{w b_{j}}-\mathbf{p}_{w b_{i}}-\mathbf{v}_{i}^{w} \Delta t+\frac{1}{2} \mathbf{g}^{w} \Delta t^{2}\right)-\boldsymbol{\alpha}_{b_{i} b_{j}}} \\ {2\left[\mathbf{q}_{b_{j} b_{i}} \otimes\left(\mathbf{q}_{b_{i} w} \otimes \mathbf{q}_{w b_{j}}\right)\right]_{x y z}} \\ {\mathbf{q}_{b_{i} w}\left(\mathbf{v}_{j}^{w}-\mathbf{v}_{i}^{w}+\mathbf{g}^{w} \Delta t\right)-\beta_{b_{i} b_{j}}} \\ {\mathbf{b}_{j}^{a}-\mathbf{b}_{i}^{a}} \\ {\mathbf{b}_{j}^{g}-\mathbf{b}_{i}^{g}}\end{array}\right]
$$

> 在vins中，这个残差块与i帧时刻位姿、速度、bias还有j帧时刻的位姿、速度、bias相关，也就是最小化这个残差块需要不断优化以上的参数块。

### imu残差的雅克比
 
> 此处我们发现，上面的imu残差一共有五维，因此在计算雅克比时，可以对五个维度的残差分别对待优化参数块进行求导。由于此处如果全部进行求导，实在会太多公式，因此，此处仅仅以$$r_v$$为例，列举几个求导过程

- $$r_v$$对i时刻位置求导

$$
\frac{\partial \mathbf{r}_{v}}{\partial \delta \mathbf{p}_{b_{i} b_{i}^{\prime}}}=\mathbf{0}
$$

- $$r_v$$对i时刻角度求导

$$
\begin{aligned} \frac{\partial \mathbf{r}_{v}}{\partial \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}} &=\frac{\partial\left(\mathbf{q}_{w b_{i}} \otimes\left[\begin{array}{c}{1} \\ {\frac{1}{2} \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}}\end{array}\right]\right)^{-1}\left(\mathbf{v}_{j}^{w}-\mathbf{v}_{i}^{w}+\mathbf{g}^{w} \Delta t\right)}{\partial \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}} \\ &=\frac{\partial\left(\mathbf{R}_{w b_{i}} \exp \left(\left[\delta \boldsymbol{\theta}_{b_{i}}^{\prime}\right] \times\right)\right)^{-1}\left(\mathbf{v}_{j}^{w}-\mathbf{v}_{i}^{w}+\mathbf{g}^{w} \Delta t\right)}{\partial \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}} \\ &=\frac{\partial \exp \left(\left[-\delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}\right]_{\mathbf{X}}\right) \mathbf{R}_{b_{i} w}\left(\mathbf{v}_{j}^{w}-\mathbf{v}_{i}^{w}+\mathbf{g}^{w} \Delta t\right)}{\partial \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}} \end{aligned}
$$

$$
\begin{aligned} &=\frac{\partial\left(\mathbf{I}-\left[\delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}\right] \times\right) \mathbf{R}_{b_{i} w}\left(\mathbf{v}_{j}^{w}-\mathbf{v}_{i}^{w}+\mathbf{g}^{w} \Delta t\right)}{\partial \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}} \\=& \frac{\partial-\left[\delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}\right] \times \boldsymbol{R}_{b_{i} w}\left(\mathbf{v}_{j}^{w}-\mathbf{v}_{i}^{w}+\mathbf{g}^{w} \Delta t\right)}{\partial \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}} \\=& \frac{\partial\left[\mathbf{R}_{b_{i} w}\left(\mathbf{v}_{j}^{w}-\mathbf{v}_{i}^{w}+\mathbf{g}^{w} \Delta t\right]_{ \times}\right) \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}}{\partial \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}} \\=&\left[\mathbf{R}_{b_{i} w}\left(\mathbf{v}_{j}^{w}-\mathbf{v}_{i}^{w}+\mathbf{g}^{w} \Delta t\right)\right]_{\mathbf{x}} \end{aligned}
$$

- $$r_v$$对i时刻速度求导

$$
\frac{\partial \mathbf{r}_{v}}{\partial \delta \mathbf{v}_{i}^{w}}=-\mathbf{R}_{b_{i} w}
$$

- $$r_v$$对i时刻bias求导（注意，此处预积分对bias的导数在初始化时就已经求过了）

$$
\frac{\partial \mathbf{r}_{v}}{\partial \delta \mathbf{b}_{i}^{a}}=-\frac{\partial \boldsymbol{\beta}_{b_{i} b_{j}}}{\partial \delta \mathbf{b}_{i}^{a}}=-\mathbf{J}_{b_{i}^{a}}^{\beta}
$$

> 其实在计算imu残差的雅克比矩阵时，会多次用到链式法则，因为在另外一篇讲解[imu预积分](http://xhy3054.github.io/triangulation/)的博客中，已经计算出了imu预积分相对于bias的雅克比。

### imu残差的协方差
上文中给出了残差与雅克比的计算，我们知道如果想使用高斯牛顿或者LM算法，最好还需要给出准确的协方差。此处由于在imu积分中，协方差是一直传递的。我们知道imu增量的递推公式可以写为：

$$\delta z_{k+1}^{15 \times 1}=F^{15 \times 15} \delta z_{k}^{15 \times 1} + V^{15 \times 18} n^{18 \times 1}$$

由上式我们知道，协方差的传递公式为：

$$ P_{k+1} = F P_k F^T + V Q V^T, P_0 = 0 $$

此处我们假设初始值是为0，噪声项的**对角协方差矩阵Q**可以通过艾伦方差标定得到(此处前两项是k时刻的加速度与角速度的高斯白噪方差，中间两项是k+1时刻的高斯白噪方差，最后两项是k时刻的bias随机游走方差，它们之间互相不相关，所以是对角矩阵)。

$$ Q^{18 \times 18} = \left(\sigma_{a}^2 , \sigma_{w}^2 , \sigma_{a}^2 , \sigma_{w}^2 , \sigma_{ba}^2 , \sigma_{bw}^2 \right) $$


## 视觉观测的重投影误差
视觉残差也就是特征点的重投影误差，vins中的视觉残差是由每个特征点从观测到它的第一帧，到所有可以观测到它的其他帧的重投影误差组成的。

拿其中任意一项举例，比如这项是一个特征点从i帧到j帧的重投影误差，这个特征点在i帧相机平面的归一化坐标为$$(u_{ci}, v_{ci})$$，深度为$$\lambda$$，在j帧相机平面的归一化坐标为$$(u_{cj}, v_{cj})$$。假设将这个点从i帧投影到j帧的相机坐标系，则过程如下：

$$
\left[\begin{array}{c}{x_{c_{j}}} \\ {y_{c_{j}}} \\ {z_{c_{j}}} \\ {1}\end{array}\right]=\mathbf{T}_{b c}^{-1} \mathbf{T}_{w b_{j}}^{-1} \mathbf{T}_{w b_{i}} \mathbf{T}_{b c}\left[\begin{array}{c}{\frac{1}{\lambda} u_{c_{i}}} \\ {\frac{1}{\lambda} v_{c_{i}}} \\ {\frac{1}{\lambda}} \\ {1}\end{array}\right]
$$

整个投影过程比较长，依次是将特征点

1. 从i帧相机坐标系变换到i帧body坐标系

2. 从i帧body坐标系变换到世界坐标系

3. 从世界坐标系变换到j帧body坐标系

4. 从j帧body坐标系变换到j帧相机坐标系

将上面这个变换拆分成3维坐标形式，变换形式如下：

$$
\begin{aligned} \mathbf{f}_{c_{j}}=\left[\begin{array}{c}{x_{c_{j}}} \\ {y_{c_{j}}} \\ {z_{c_{j}}}\end{array}\right] &=\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}} \mathbf{R}_{b c} \frac{1}{\lambda}\left[\begin{array}{c}{u_{c_{i}}} \\ {v_{c_{i}}} \\ {1}\end{array}\right] \\ &+\mathbf{R}_{b c}^{\top}\left(\mathbf{R}_{w b_{j}}^{\top}\left(\left(\mathbf{R}_{w b_{i}} \mathbf{p}_{b c}+\mathbf{p}_{w b_{i}}\right)-\mathbf{p}_{w b_{j}}\right)-\mathbf{p}_{b c}\right) \end{aligned}
$$

而视觉残差就是

$$
\mathbf{r}_{c}=\left[\begin{array}{c}{\frac{x_{c_{j}}}{z_{c_{j}}}-u_{c_{j}}} \\ {\frac{y_{c_{j}}}{z_{c_{j}}}-v_{c_{j}}}\end{array}\right]
$$

### 视觉残差的雅克比

> 此处推导视觉残差的雅克比遵循了链式法则，首先求视觉残差对$$\mathbf{f}_{c_{j}}$$的导数，然后由$$\mathbf{f}_{c_{j}}$$开始求其对待优化参数的偏导数。

- 首先，残差对$$\mathbf{f}_{c_{j}}$$求导；

$$
\frac{\partial \mathbf{r}_{c}}{\partial \mathbf{f}_{c_{j}}}=\left[\begin{array}{ccc}{\frac{1}{z_{c_{j}}}} & {0} & {-\frac{x_{c_{j}}}{z_{c_{j}}^{2}}} \\ {0} & {\frac{1}{z_{c_{j}}}} & {-\frac{y_{c_{j}}}{z_{c_{j}}^{2}}}\end{array}\right]
$$

- 其次，$$\mathbf{f}_{c_{j}}$$对各状态量求导；

- $$\mathbf{f}_{c_{j}}$$对i时刻位置求导；

$$
\frac{\partial \mathbf{f}_{c_{j}}}{\partial \delta \mathbf{p}_{b_{i} b_{i}^{\prime}}}=\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top}
$$

- $$\mathbf{f}_{c_{j}}$$对i时刻姿态求导；

$$
\mathbf{f}_{c_{j}}=\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}} \mathbf{R}_{b c} \mathbf{f}_{c_{i}}+\mathbf{R}_{b c}^{\top}\left(\mathbf{R}_{w b_{i}}^{\top}\left(\left(\mathbf{R}_{w b_{i} \mathbf{p}_{b c}}+\mathbf{p}_{w b_{i}}\right)-\mathbf{p}_{w b_{j}}\right)-\mathbf{p}_{b c}\right)
$$

$$
\begin{aligned} \frac{\partial \mathbf{f}_{c_{j}}}{\partial \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}} &=\frac{\partial \mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}}\left(\mathbf{I}+\left[\delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}\right]_{\mathbf{x}}\right) \mathbf{f}_{b_{i}}}{\partial \delta \boldsymbol{\theta}_{b_{i} b_{i}^{\prime}}} \\ &=-\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}}\left[\mathbf{f}_{b_{i}}\right]_{ \times} \end{aligned}
$$

- $$\mathbf{f}_{c_{j}}$$对j时刻位置求导；

$$
\frac{\partial \mathbf{f}_{c_{j}}}{\partial \delta \mathbf{p}_{b_{j} b_{j}^{\prime}}}= - \mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top}
$$

- $$\mathbf{f}_{c_{j}}$$对j时刻姿态求导；

$$
\begin{aligned} \mathbf{f}_{c_{j}} &=\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}} \mathbf{R}_{b c} \mathbf{f}_{c_{i}}+\mathbf{R}_{b c}^{\top}\left(\mathbf{R}_{w b_{j}}^{\top}\left(\left(\mathbf{R}_{w b_{i} \mathbf{p}_{b c}}+\mathbf{p}_{w b_{i}}\right)-\mathbf{p}_{w b_{j}}\right)-\mathbf{p}_{b c}\right) \\ &=\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top}\left(\mathbf{R}_{w b_{i}}\left(\mathbf{R}_{b c} \mathbf{f}_{c_{i}}+\mathbf{p}_{b c}\right)+\mathbf{p}_{w b_{i}}-\mathbf{p}_{w b_{j}}\right)+(\ldots) \\ &=\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{i}}^{\top}\left(\mathbf{f}_{w}-\mathbf{p}_{w b_{j}}\right)+(\ldots) \end{aligned}
$$

$$
\begin{aligned} \frac{\partial \mathbf{f}_{c_{j}}}{\partial \delta \boldsymbol{\theta}_{b_{j} b_{j}^{\prime}}} &=\frac{\partial \mathbf{R}_{b c}^{\top}\left(\mathbf{I}-\left[\delta \boldsymbol{\theta}_{b_{j} b_{j}^{\prime}}\right] \times\right) \mathbf{R}_{w b_{j}}^{\top}\left(\mathbf{f}_{w}-\mathbf{p}_{w b_{j}}\right)}{\partial \delta \boldsymbol{\theta}_{b_{j} b_{j}^{\prime}}} \\ &=\frac{\partial \mathbf{R}_{b c}^{\top}\left(\mathbf{I}-\left[\delta \boldsymbol{\theta}_{b_{j} b_{j}^{\prime}}\right] \times\right) \mathbf{f}_{b_{j}}}{\partial \delta \boldsymbol{\theta}_{b_{j} b_{j}^{\prime}}} \\ &=\mathbf{R}_{b c}^{\top}\left[\mathbf{f}_{b_{j}}\right]_{ \times} \end{aligned}
$$

- $$\mathbf{f}_{c_{j}}$$对imu与相机之间的外参位移求导；

$$
\frac{\partial \mathbf{f}_{c_{j}}}{\partial \delta \mathbf{p}_{c c^{\prime}}}=\mathbf{R}_{b c}^{\top}\left(\mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}}-\mathbf{I}_{3 \times 3}\right)
$$

- $$\mathbf{f}_{c_{j}}$$对imu与相机之间的外参角度求导；

$$
\frac{\partial \mathbf{f}_{c_{j}}}{\partial \delta \boldsymbol{\theta}_{c c^{\prime}}}=-\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}} \mathbf{R}_{b c}\left[\mathbf{f}_{c_{i}}\right]_{\mathbf{x}}+\left[\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}} \mathbf{R}_{b c} \mathbf{f}_{c_{i}}\right]_{ \times} + 
\left[\mathbf{R}_{b c}^{\top}\left(\mathbf{R}_{w b_{j}}^{\top}\left(\left(\mathbf{R}_{w b_{i}} \mathbf{p}_{b c}+\mathbf{p}_{w b_{i}}\right)-\mathbf{p}_{w b_{j}}\right)-\mathbf{p}_{b c}\right)\right] \times
$$

- $$\mathbf{f}_{c_{j}}$$对特征在i帧相机坐标系中的逆深度求导；

$$
\begin{aligned} \frac{\partial \mathbf{f}_{c_{j}}}{\partial \delta \lambda} &=\frac{\partial \mathbf{f}_{c_{j}}}{\partial \mathbf{f}_{c_{i}}} \frac{\partial \mathbf{f}_{c_{i}}}{\partial \delta \lambda} \\ &=\mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}} \mathbf{R}_{b c}\left(-\frac{1}{\lambda^{2}}\left[\begin{array}{c}{u_{c_{i}}} \\ {v_{c_{i}}} \\ {1}\end{array}\right]\right) \\ &=-\frac{1}{\lambda} \mathbf{R}_{b c}^{\top} \mathbf{R}_{w b_{j}}^{\top} \mathbf{R}_{w b_{i}} \mathbf{R}_{b c} \mathbf{f}_{c_{i}} \end{aligned}
$$

> 上面的推导公式中，有一些推导步骤实在太过繁琐复杂，因此被我舍去了，如果大家想看，可以自己手动推导或者查看一些vio论文，会有详细推导过程。

### 视觉约束的协方差
在vins代码中假设重投影到像素平面上时的`sqrt_info`为1.5个像素，对应到归一化平面上需要除以焦距f。这个是假设视觉投影时投影点的位置在理论值附近呈均值为0，标准差为1.5个像素的高斯分布。代码部分如下：

    ProjectionFactor::sqrt_info = FOCAL_LENGTH / 1.5 * Matrix2d::Identity();


## 先验残差

## marg掉最老的关键帧

## marg掉次新的关键帧




