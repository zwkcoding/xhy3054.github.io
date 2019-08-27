---
layout: post
title: vins初始化中的陀螺仪偏置标定
date: 2019-08-23 10:07:24.000000000 +09:00
img:  one-piece/one-piece13.jpg # Add image post (optional)
tag: [多传感器融合]
---

vins是一个很好的开源工作，最近一直在熟悉它，其中大部分的内容之前已经或多多少提到过了，这篇博客主要讲一讲vins初始化中的视觉与imu联合初始化工作。

> 注意：$$T_{WI}$$指的是从I系到W系的变换矩阵。

# 陀螺仪偏置标定问题
> 这个工作主要是在`vins/vins_estimator/src/initial/initial_aligment.cpp`文件中的`solveGyroscopeBias`函数中实现的。

对于窗口中连续两帧图像$b_{k}$和$b_{k+1}$，之前已经通过视觉sfm得到了两帧相对于滑窗第一帧的旋转$$\mathbf{q}_{c_{0} b_{k}}$$和$$\mathbf{q}_{c_{0} b_{k+1}}$$，此时之前通过IMU预积分得到这两帧旋转的预积分$$\hat{\gamma}_{b_k b_{k+1}}$$（先前不带bias的预积分）。

此时，我们可以建立约束方程，最小化代价函数，如下：

$$
\arg \min _{\delta \mathbf{b}^{g}} \sum_{k \in B}\left\|2\left\lfloor\mathbf{q}_{c_{0} b_{k+1}}^{-1} \otimes \mathbf{q}_{c_{0} b_{k}} \otimes {\gamma}_{b_k b_{k+1}}\right\rfloor_{x y z}\right\|^{2}
$$

> 其中$${\gamma}_{b_k b_{k+1}}$$表示的是算上陀螺仪bias的预积分值。

$$
\gamma_{b_k b_{k+1}} \approx \hat{\gamma}_{b_k b_{k+1}} \otimes\left[\begin{array}{c}{1} \\ {\frac{1}{2} J_{b_{g}}^{\gamma} \delta b_{g}}\end{array}\right]
$$

## 具体方法
在上述问题的求解时，最理想的情况如下（其中等式右边的四元数代表的是旋转不变的四元数）：

$$
\mathbf{q}_{c_{0} b_{k+1}}^{-1} \otimes \mathbf{q}_{c_{0} b_{k}} \otimes {\gamma}_{b_k b_{k+1}} = \left[\begin{array}{l}{1} \\ {0}\end{array}\right]
$$

对上式进行变换可以得到

$$
{\gamma}_{b_k b_{k+1}} = \mathbf{q}_{c_{0} b_{k}}^{-1} \otimes \mathbf{q}_{c_{0} b_{k+1}} \otimes \left[\begin{array}{l}{1} \\ {0}\end{array}\right]
$$

将$$\delta b_{g}$$带入上式可以得到

$$
\hat{\gamma}_{b_k b_{k+1}} \otimes\left[\begin{array}{c}{1} \\ {\frac{1}{2} J_{b_{g}}^{\gamma} \delta b_{g}}\end{array}\right] = \mathbf{q}_{c_{0} b_{k}}^{-1} \otimes \mathbf{q}_{c_{0} b_{k+1}} \otimes \left[\begin{array}{l}{1} \\ {0}\end{array}\right]
$$

只考虑上式虚部，我们可以得到

$$
J_{b_{g}}^{\gamma} \delta b_{g}=2\left(\hat{\gamma}_{b_k b_{k+1}} \otimes \mathbf{q}_{c_{0} b_{k}}^{-1} \otimes \mathbf{q}_{c_{0} b_{k+1}}\right)_{v e c}
$$

对于上式，我们想要做的是求解出$$\delta b_{g}$$的大小，此时我们可以使用如下办法：

- 等式两侧同时乘以$$J_{b_{g}}^{\gamma T}$$，即得到

$$
J_{b_{g}}^{\gamma T} J_{b_{g}}^{\gamma} \delta b_{g}=2 J_{b_{g}}^{\gamma T} \left(\hat{\gamma}_{b_k b_{k+1}} \otimes \mathbf{q}_{c_{0} b_{k}}^{-1} \otimes \mathbf{q}_{c_{0} b_{k+1}}\right)_{v e c}
$$

- 使用LDLT分解即可求得 $$\delta b_{g}$$

## vins中代码实现
```cpp
/**
 * @brief   陀螺仪偏置校正
 * @optional    根据视觉SFM的结果来校正陀螺仪Bias -> Paper V-B-1
 *              主要是将相邻帧之间SFM求解出来的旋转矩阵与IMU预积分的旋转量对齐
 *              注意得到了新的Bias后对应的预积分需要repropagate
 * @param[in]   all_image_frame所有图像帧构成的map,图像帧保存了位姿、预积分量和关于角点的信息
 * @param[out]  Bgs 陀螺仪偏置
 * @return      void
*/
void solveGyroscopeBias(map<double, ImageFrame> &all_image_frame, Vector3d* Bgs)
{
    Matrix3d A;
    Vector3d b;
    Vector3d delta_bg;
    A.setZero();
    b.setZero();
    map<double, ImageFrame>::iterator frame_i;
    map<double, ImageFrame>::iterator frame_j;
    //遍历所有图像帧
    for (frame_i = all_image_frame.begin(); next(frame_i) != all_image_frame.end(); frame_i++)
    {
        //下一帧
        frame_j = next(frame_i);
        MatrixXd tmp_A(3, 3);
        tmp_A.setZero();
        VectorXd tmp_b(3);
        tmp_b.setZero();

        //两帧之间的旋转变换 
        //R_ij = (R^c0_bk)^-1 * (R^c0_bk+1) 转换为四元数 q_ij = (q^c0_bk)^-1 * (q^c0_bk+1)
        Eigen::Quaterniond q_ij(frame_i->second.R.transpose() * frame_j->second.R);
        //tmp_A = J_j_bw 雅可比 ，之前的预积分是没有算上陀螺仪bias的
        tmp_A = frame_j->second.pre_integration->jacobian.template block<3, 3>(O_R, O_BG);
        //tmp_b = 2 * (r^bk_bk+1)^-1 * (q^c0_bk)^-1 * (q^c0_bk+1)
        //      = 2 * (r^bk_bk+1)^-1 * q_ij
        tmp_b = 2 * (frame_j->second.pre_integration->delta_q.inverse() * q_ij).vec();
        //tmp_A * delta_bg = tmp_b
        A += tmp_A.transpose() * tmp_A;
        b += tmp_A.transpose() * tmp_b;

    }
    //LDLT方法
    delta_bg = A.ldlt().solve(b);
    ROS_WARN_STREAM("gyroscope bias initial calibration " << delta_bg.transpose());

    //假设窗口中bgs值都相同
    for (int i = 0; i <= WINDOW_SIZE; i++)
        Bgs[i] += delta_bg;

    //标定成功bias后需要重新进行预积分
    for (frame_i = all_image_frame.begin(); next(frame_i) != all_image_frame.end( ); frame_i++)
    {
        frame_j = next(frame_i);
        frame_j->second.pre_integration->repropagate(Vector3d::Zero(), Bgs[0]);
    }
}

```
---
