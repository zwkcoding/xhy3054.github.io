---
layout: post
title: IMU预积分
date: 2019-07-31 10:07:24.000000000 +09:00
img:  one-piece/one-piece12.jpg # Add image post (optional)
tag: [多传感器融合]
---

在vio中会经常看到预积分这个词，vins中有，okvis中也有，所以，这个东西到底有什么作用呢？

# 预积分的由来
首先，一般imu的频率是比相机高的，所以在两个图像帧之间，会有很多imu采集的数据。如下图：
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/vio/pre-inte.PNG"/>
</div>

此处假设IMU的真实值是$\omega, \mathbf{a}$，测量值为$\tilde{\boldsymbol{\omega}}, \tilde{\mathbf{a}}$，则有：

$$
\begin{aligned} \tilde{\omega}^{b} &=\omega^{b}+\mathbf{b}^{g}+\mathbf{n}^{g} \\ \tilde{\mathbf{a}}^{b} &=\mathbf{q}_{b w}\left(\mathbf{a}^{w}+\mathbf{g}^{w}\right)+\mathbf{b}^{a}+\mathbf{n}^{a} \end{aligned}
$$

其中上标$g$表示gyro（陀螺仪坐标系），$a$表示acc（加速度计坐标系），$w$表示在世界坐标系world，$b$表示imu机体坐标系body。


根据运动学方程，位置、速度、姿态（PVQ）的导数有如下：

$$
\begin{array}{l}{\dot{\mathbf{p}}_{w b_{t}}=\mathbf{v}_{t}^{w}} \\ {\dot{\mathbf{v}}_{t}^{w}=\mathbf{a}_{t}^{w}} \\ {\dot{\mathbf{q}}_{w b_{t}}=\mathbf{q}_{w b_{t}} \otimes\left[\begin{array}{c}{0} \\ {\frac{1}{2} \boldsymbol{\omega}^{b_{t}}}\end{array}\right]}\end{array}
$$

从第i时刻的PVQ对IMU的测量值进行积分得到第j时刻的PVQ：

$$
\begin{array}{l}{\mathbf{p}_{w b_{j}}=\mathbf{p}_{w b_{i}}+\mathbf{v}_{i}^{w} \Delta t+\iint_{t \in[i, j]}\left(\mathbf{q}_{w b_{t}} \mathbf{a}^{b_{t}}-\mathbf{g}^{w}\right) \delta t^{2}} \\ {\mathbf{v}_{j}^{w}=\mathbf{v}_{i}^{w}+\int_{t \in[i, j]}\left(\mathbf{q}_{w b_{t}} \mathbf{a}^{b_{t}}-\mathbf{g}^{w}\right) \delta t} \\ {\mathbf{q}_{w b_{j}}=\int_{t \in[i, j]} \mathbf{q}_{w b_{t}} \otimes\left[\begin{array}{c}{0} \\ {\frac{1}{2} \boldsymbol{\omega}^{b_{t}} ] \delta t}\end{array}\right.}\end{array}
$$

> 此时我们发现，在积分项中存在$q_{wb_{t}}$，同时在$\mathbf{a}^{b_{t}}$等测量值中还会引入bias。这样每次在迭代优化过后，由于$q_{wb_{t}}$与bias都会更新，因此此时就需要重新进行积分操作。这个计算量是非常巨大的。

## 预积分消减计算量
为了消减计算量，有人提出了通过如下的简单变换，**将上述积分模型转换成预积分模型**。这个变换是：

$$
\mathbf{q}_{w b_{t}}=\mathbf{q}_{w b_{i}} \otimes \mathbf{q}_{b_{i} b_{t}}
$$

转换后，整个PVQ的积分形式变为：

$$
\begin{array}{l}{\mathbf{p}_{w b_{j}}=\mathbf{p}_{w b_{i}}+\mathbf{v}_{i}^{w} \Delta t-\frac{1}{2} \mathbf{g}^{w} \Delta t^{2}+\mathbf{q}_{w b_{i}} \iint_{t \in[i, j]}\left(\mathbf{q}_{b_{i} b_{t}} \mathbf{a}^{b_{t}}\right) \delta t^{2}} \\ {\mathbf{v}_{j}^{w}=\mathbf{v}_{i}^{w}-\mathbf{g}^{w} \Delta t+\mathbf{q}_{w b_{i}} \int_{t \in[i, j]}\left(\mathbf{q}_{b_{i} b_{t}} \mathbf{a}^{b_{t}}\right) \delta t} \\ {\mathbf{q}_{w b_{j}}=\mathbf{q}_{w b_{i}} \int_{t \in[i, j]} \mathbf{q}_{b_{i} b_{t}} \otimes\left[\begin{array}{c}{0} \\ {\frac{1}{2} \boldsymbol{\omega}^{b_{t}}}\end{array}\right] \delta t}\end{array}
$$

上面公式中，我们可以将其中一部分提取出来，叫做**预积分量**：

$$
\begin{aligned} \boldsymbol{\alpha}_{b_{i} b_{j}} &=\iint_{t \in[i, j]}\left(\mathbf{q}_{b_{i} b_{t}} \mathbf{a}^{b_{t}}\right) \delta t^{2} \\ \boldsymbol{\beta}_{b_{i} b_{j}} &=\int_{t \in[i, j]}\left(\mathbf{q}_{b_{i} b_{t}} \mathbf{a}^{b_{t}}\right) \delta t \\ \mathbf{q}_{b_{i} b_{j}} &=\int_{t \in[i, j]} \mathbf{q}_{b_{i} b_{t}} \otimes\left[\begin{array}{c}{0} \\ {\frac{1}{2} \boldsymbol{\omega}^{b_{t}}}\end{array}\right] \delta t \end{aligned}
$$

> 如上，我们发现，**预积分量**只与IMU的测量有关，如果不考虑更新IMU的bias，那么我们只需要进行一次预积分，以后就无需在重新计算了。

# vins中预积分的使用
我们知道，在vins的非线性优化中不止进行了相机姿态还有路标的优化，也将imu的bias加入了优化的变量中。这样在一次优化更新后，其实bias是会发生改变的，这样预积分量也就需要更新，那么积分就还需要重新计算？

其实在vins中为了解决这个问题，是这样做的(当更新前后bias变化比较小时)：

$$
\begin{aligned} \boldsymbol{\alpha}_{b_{k+1}}^{b_{k}} & \approx \hat{\boldsymbol{\alpha}}_{b_{k+1}}^{b_{k}}+\mathbf{J}_{b_{a}}^{\alpha} \delta \mathbf{b}_{a_{k}}+\mathbf{J}_{b_{w}}^{\alpha} \delta \mathbf{b}_{w_{k}} \\ \boldsymbol{\beta}_{b_{k+1}}^{b_{k}} & \approx \hat{\boldsymbol{\beta}}_{b_{k+1}}^{b_{k}}+\mathbf{J}_{b_{a}}^{\beta} \delta \mathbf{b}_{a_{k}}+\mathbf{J}_{b_{w}}^{\beta} \delta \mathbf{b}_{w_{k}} \\ \boldsymbol{\gamma}_{b_{k+1}}^{b_{k}} & \approx \hat{\boldsymbol{\gamma}}_{b_{k+1}}^{b_{k}} \otimes\left[\begin{array}{c}{1} \\ {\frac{1}{2} \mathbf{J}_{b_{w}}^{\gamma} \delta \mathbf{b}_{w_{k}}}\end{array}\right] \end{aligned}
$$

> 当一次非线性优化后，**如果bias的变化比较小**，会使用一阶展开来近似更新**从k帧到k+1帧**的预积分量。**当bias变化比较大时**，近似效果会不好，此时需要重新进行积分。（此处假设使用欧拉积分）这样做可以很大的消减基于优化的方法的计算量。


# 参考文献
- [1] 深蓝学院vio课程讲义
- [2] Leutenegger S, Furgale P, Rabaud V, et al. Keyframe-based visual-inertial slam using nonlinear optimization[J]. Proceedings of Robotis Science and Systems (RSS) 2013, 2013.
- [3] Qin T, Li P, Shen S. Vins-mono: A robust and versatile monocular visual-inertial state estimator[J]. IEEE Transactions on Robotics, 2018, 34(4): 1004-1020.
