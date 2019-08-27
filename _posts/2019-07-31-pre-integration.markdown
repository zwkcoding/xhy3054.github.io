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

> 此时我们发现，在积分项中存在全局姿态$q_{wb_{t}}$，同时在$\mathbf{a}^{b_{t}}$等测量值中还会引入bias。这样每次在迭代优化过后，由于$q_{wb_{t}}$与bias都会更新，因此此时就需要重新进行积分操作。这个计算量是非常巨大的。

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

> 如上，我们发现，其实预积分就是将原本的积分变成了相邻两帧中相对运动的积分。这使得**预积分量**只与IMU的测量有关，如果不考虑更新IMU的bias，那么我们只需要进行一次预积分，以后就无需在重新计算了。

## 基于预积分的状态变换

此时状态量（世界位姿与bias等）和预积分的映射关系如下：

$$
\left[\begin{array}{c}{\mathbf{p}_{w b_{j}}} \\ {\mathbf{v}_{j}^{w}} \\ {\mathbf{q}_{w b_{j}}} \\ {\mathbf{b}_{j}^{a}} \\ {\mathbf{b}_{j}^{g}}\end{array}\right]=\left[\begin{array}{cc}{\mathbf{p}_{w b_{i}}+\mathbf{v}_{i}^{w} \Delta t-\frac{1}{2} \mathbf{g}^{w} \Delta t^{2}+\mathbf{q}_{w b_{i}} \mathbf{\alpha}_{b_{i} b_{j}}} \\ {\mathbf{v}_{i}^{w}-\mathbf{g}^{w} \Delta t+\mathbf{q}_{w b_{i}} \boldsymbol{\beta}_{b_{i} b_{j}}} \\ {\mathbf{q} w b_{i} \mathbf{q}_{b i}} \\ {\mathbf{b}_{i}^{a}} \\ {\mathbf{b}_{i}^{g}}\end{array}\right]
$$

此时，如果我们想要建立基于目前状态的预积分误差，则是如下形式（其中表示位姿的四元数我们取其虚部并乘以2恢复出旋转向量）：

$$
\left[\begin{array}{c}{\mathbf{r}_{p}} \\ {\mathbf{r}_{q}} \\ {\mathbf{r}_{v}} \\ {\mathbf{r}_{b a}} \\ {\mathbf{r}_{b g}}\end{array}\right]_{15 \times 1}=\left[\begin{array}{c}{\mathbf{q}_{b_{i} w}\left(\mathbf{p}_{w b_{j}}-\mathbf{p}_{w b_{i}}-\mathbf{v}_{i}^{w} \Delta t+\frac{1}{2} \mathbf{g}^{w} \Delta t^{2}\right)-\boldsymbol{\alpha}_{b_{i} b_{j}}} \\ {2\left[\mathbf{q}_{b_{j} b_{i}} \otimes\left(\mathbf{q}_{b_{i} w} \otimes \mathbf{q}_{w b_{j}}\right)\right]_{x y z}} \\ {\mathbf{q}_{b_{i} w}\left(\mathbf{v}_{j}^{w}-\mathbf{v}_{i}^{w}+\mathbf{g}^{w} \Delta t\right)-\beta_{b_{i} b_{j}}} \\ {\mathbf{b}_{j}^{a}-\mathbf{b}_{i}^{a}} \\ {\mathbf{b}_{j}^{g}-\mathbf{b}_{i}^{g}}\end{array}\right]
$$

由于k+1帧段的预积分受其他变量的影响，为了以后方便处理，可以推导出其雅克比矩阵，整体形式如下：

$$
\left[\begin{array}{c}{\delta \boldsymbol{\alpha}_{b_{k+1} b_{k+1}^{\prime}}} \\ {\delta \boldsymbol{\theta}_{b_{k+1} b_{k+1}^{\prime}}} \\ {\delta \boldsymbol{\beta}_{b_{k+1} b_{k+1}^{\prime}}} \\ {\delta \mathbf{b}_{k+1}^{a}} \\ {\delta \mathbf{b}_{k+1}^{g}}\end{array}\right]=\mathbf{F}\left[\begin{array}{c}{\delta \boldsymbol{\alpha}_{b_{k} b_{k}^{\prime}}} \\ {\delta \boldsymbol{\theta}_{b_{k} b_{k}^{\prime}}} \\ {\delta \boldsymbol{\beta}_{b_{k} b_{k}^{\prime}}} \\ {\delta \mathbf{b}_{k}^{a}} \\ {\delta \mathbf{b}_{k}^{g}}\end{array}\right]+\mathbf{G}\left[\begin{array}{c}{\mathbf{n}_{k}^{a}} \\ {\mathbf{n}_{k}^{g}} \\ {\mathbf{n}_{k+1}^{a}} \\ {\mathbf{n}_{k+1}^{g}} \\ {\mathbf{n}_{\mathbf{b}_{k}^{a}}} \\ {\mathbf{n}_{\mathbf{b}_{k}^{g}}^{g}}\end{array}\right]
$$

其中部分雅克比系数的推导如下：

$$
\mathbf{f}_{12}=\frac{\partial \boldsymbol{\alpha}_{b_{i} b_{k+1}}}{\partial \delta \boldsymbol{\theta}_{b_{k} b_{k}^{\prime}}}=-\frac{1}{4}\left(\mathbf{R}_{b_{i} b_{k}}\left[\mathbf{a}^{b_{k}}-\mathbf{b}_{k}^{a}\right]_{ \times} \delta t^{2}+\mathbf{R}_{b_{i} b_{k+1}}\left[\left(\mathbf{a}^{b_{k+1}}-\mathbf{b}_{k}^{a}\right)\right]_{ \times}\left(\mathbf{I}-[\boldsymbol{\omega}]_{ \times} \boldsymbol{\delta} t\right) \delta t^{2}\right)
$$

$$
\mathbf{f}_{32}=\frac{\partial \boldsymbol{\beta}_{b_{i} b_{k+1}}}{\partial \delta \boldsymbol{\theta}_{b_{k} b_{k}^{\prime}}}=-\frac{1}{2}\left(\mathbf{R}_{b_{i} b_{k}}\left[\mathbf{a}^{b_{k}}-\mathbf{b}_{k}^{a}\right]_{ \times} \delta t+\mathbf{R}_{b_{i} b_{k+1}}\left[\left(\mathbf{a}^{b_{k+1}}-\mathbf{b}_{k}^{a}\right)\right]_{ \times}\left(\mathbf{I}-[\boldsymbol{\omega}]_{ \times} \delta t\right) \delta t\right)
$$

$$
\mathbf{f}_{15}=\frac{\partial \boldsymbol{\alpha}_{b_{i} b_{k+1}}}{\partial \delta \mathbf{b}_{k}^{g}}=-\frac{1}{4}\left(\mathbf{R}_{b_{i} b_{k+1}}\left[\left(\mathbf{a}^{b_{k+1}}-\mathbf{b}_{k}^{a}\right)\right] \times \delta t^{2}\right)(-\delta t)
$$

$$
\mathbf{f}_{35}=\frac{\partial \boldsymbol{\beta}_{b_{i} b_{k+1}}}{\partial \delta \mathbf{b}_{k}^{g}}=-\frac{1}{2}\left(\mathbf{R}_{b_{i} b_{k+1}}\left[\left(\mathbf{a}^{b_{k+1}}-\mathbf{b}_{k}^{a}\right)\right] \times \delta t\right)(-\delta t)
$$

$$
\mathbf{g}_{12}=\frac{\partial \boldsymbol{\alpha}_{b_{i} b_{k+1}}}{\partial \mathbf{n}_{k}^{g}}=\mathbf{g}_{14}=\frac{\partial \boldsymbol{\alpha}_{b_{i} b_{k+1}}}{\partial \mathbf{n}_{k+1}^{g}}=-\frac{1}{4}\left(\mathbf{R}_{b_{i} b_{k+1}}\left[\left(\mathbf{a}^{b_{k+1}}-\mathbf{b}_{k}^{a}\right)\right] \times \delta t^{2}\right)\left(\frac{1}{2} \delta t\right)
$$

$$
\mathbf{g}_{32}=\frac{\partial \boldsymbol{\beta}_{b_{i} b_{k+1}}}{\partial \mathbf{n}_{k}^{g}}=\mathbf{g}_{34}=\frac{\partial \boldsymbol{\beta}_{b_{i} b_{k+1}}}{\partial \mathbf{n}_{k+1}^{g}}=-\frac{1}{2}\left(\mathbf{R}_{b_{i} b_{k+1}}\left[\left(\mathbf{a}^{b_{k+1}}-\mathbf{b}_{k}^{a}\right)\right] \times \delta t\right)\left(\frac{1}{2} \delta t\right)
$$

## 基于中值积分的预积分
首先，对于姿态变化的中值积分首先进行

$$
\begin{aligned} \boldsymbol{\omega} &=\frac{1}{2}\left(\left(\boldsymbol{\omega}^{b_{k}}+\mathbf{n}_{k}^{g}-\mathbf{b}_{k}^{g}\right)+\left(\boldsymbol{\omega}^{b_{k+1}}+\mathbf{n}_{k+1}^{g}-\mathbf{b}_{k}^{g}\right)\right) \\ \mathbf{q}_{b_{i} b_{k+1}} &=\mathbf{q}_{b_{i} b_{k}} \otimes\left[\begin{array}{c}{1} \\ {\frac{1}{2} \boldsymbol{\omega} \delta t}\end{array}\right] \end{aligned}
$$

然后，会完成位置变化的中值积分

$$
\begin{aligned} \mathbf{a} &=\frac{1}{2}\left(\mathbf{q}_{b_{i} b_{k}}\left(\mathbf{a}^{b_{k}}+\mathbf{n}_{k}^{a}-\mathbf{b}_{k}^{a}\right)+\mathbf{q}_{b_{i} b_{k+1}}\left(\mathbf{a}^{b_{k+1}}+\mathbf{n}_{k+1}^{a}-\mathbf{b}_{k}^{a}\right)\right) \\ \boldsymbol{\alpha}_{b_{i} b_{k+1}} &=\boldsymbol{\alpha}_{b_{i} b_{k}}+\boldsymbol{\beta}_{b_{i} b_{k}} \delta t+\frac{1}{2} \mathbf{a} \delta t^{2} \\ \boldsymbol{\beta}_{b_{i} b_{k+1}} &=\boldsymbol{\beta}_{b_{i} b_{k}}+\mathbf{a} \delta t \end{aligned}
$$

最后是bias的更新：

$$
\begin{aligned} \mathbf{b}_{k+1}^{a} &=\mathbf{b}_{k}^{a} \\ \mathbf{b}_{k+1}^{g} &=\mathbf{b}_{k}^{g} \end{aligned}
$$

在vins中，这部分的代码如下（`vins_estimator`模块中`factor`文件夹下的头文件`integration_base.h`中的`midPointIntegration`函数）：
```cpp
        //此处预积分也就是在相邻两帧之间的相对积分，得到的是相对的位姿变换

        //ROS_INFO("midpoint integration"); 加速度中值测量前一个测量
        Vector3d un_acc_0 = delta_q * (_acc_0 - linearized_ba);
        //陀螺仪中值测量
        Vector3d un_gyr = 0.5 * (_gyr_0 + _gyr_1) - linearized_bg;
        //旋转预积分
        result_delta_q = delta_q * Quaterniond(1, un_gyr(0) * _dt / 2, un_gyr(1) * _dt / 2, un_gyr(2) * _dt / 2);
        //加速度中值测量的后一个测量
        Vector3d un_acc_1 = result_delta_q * (_acc_1 - linearized_ba);
        //加速度计中值测量
        Vector3d un_acc = 0.5 * (un_acc_0 + un_acc_1);
        //位置预积分
        result_delta_p = delta_p + delta_v * _dt + 0.5 * un_acc * _dt * _dt;
        //速度预积分
        result_delta_v = delta_v + un_acc * _dt;
        //bias默认不变（预积分时）
        result_linearized_ba = linearized_ba;
        result_linearized_bg = linearized_bg;         
```
---

# 预积分更新问题
我们知道，在vins的非线性优化中不止进行了相机姿态还有路标的优化，也将imu的bias加入了优化的变量中。这样在一次优化更新后，其实bias是会发生改变的，这样预积分量也就需要更新，那么积分就还需要重新计算？

其实在vins中为了解决这个问题，是这样做的(当更新前后bias变化比较小时)：

$$
\begin{aligned} \boldsymbol{\alpha}_{b_{k+1}}^{b_{k}} & \approx \hat{\boldsymbol{\alpha}}_{b_{k+1}}^{b_{k}}+\mathbf{J}_{b_{a}}^{\alpha} \delta \mathbf{b}_{a_{k}}+\mathbf{J}_{b_{w}}^{\alpha} \delta \mathbf{b}_{w_{k}} \\ \boldsymbol{\beta}_{b_{k+1}}^{b_{k}} & \approx \hat{\boldsymbol{\beta}}_{b_{k+1}}^{b_{k}}+\mathbf{J}_{b_{a}}^{\beta} \delta \mathbf{b}_{a_{k}}+\mathbf{J}_{b_{w}}^{\beta} \delta \mathbf{b}_{w_{k}} \\ \boldsymbol{\gamma}_{b_{k+1}}^{b_{k}} & \approx \hat{\boldsymbol{\gamma}}_{b_{k+1}}^{b_{k}} \otimes\left[\begin{array}{c}{1} \\ {\frac{1}{2} \mathbf{J}_{b_{w}}^{\gamma} \delta \mathbf{b}_{w_{k}}}\end{array}\right] \end{aligned}
$$

> 当一次非线性优化后，**如果bias的变化比较小**，会使用一阶展开来近似更新**从k帧到k+1帧**的预积分量。**当bias变化比较大时**，近似效果会不好，此时需要重新进行积分。（此处假设使用欧拉积分）这样做可以很大的消减基于优化的方法的计算量。

# vins中预积分的实现
vins中的预积分功能，主要是在`vins_estimator`模块中`factor`文件夹下的头文件`integration_base.h`中实现。

在vins程序代码中，对于每两帧图像之间都会使用一个`integrationBase`对象维护此两帧之间的预积分。其中主要有以下几个值得注意的地方：

1. 代码中对于图像与imu数据的时间戳对齐问题使用线性插值的方法进行处理（线性插值获得图像帧时间戳时的imu测量）；

2. 预积分使用中值积分进行；

3. 在预积分是可以叠加的，每次读入新的imu测量数据，就会进行一个中值预积分操作；

4. 每次中值预积分完成后，会更新其雅克比矩阵与方差矩阵（此处的雅克比矩阵在imu的bias变化后，可以直接拿来更新预积分，不用重新积分）

# 参考文献
- [1] 深蓝学院vio课程讲义
- [2] Leutenegger S, Furgale P, Rabaud V, et al. Keyframe-based visual-inertial slam using nonlinear optimization[J]. Proceedings of Robotis Science and Systems (RSS) 2013, 2013.
- [3] Qin T, Li P, Shen S. Vins-mono: A robust and versatile monocular visual-inertial state estimator[J]. IEEE Transactions on Robotics, 2018, 34(4): 1004-1020.
