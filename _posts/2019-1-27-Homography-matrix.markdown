---
layout: post
title: 射影几何中的单应变换 
date: 2019-01-27 10:07:24.000000000 +09:00
img:  game5.jpg # Add image post (optional)
tag: [图像处理]
---

## 单应矩阵
如下图所示，我们假设具有如下图所示的対极约束关系。
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/epipolar_geometry/jihe.png"/>
</div>
- 其中一空间点 X (不通过两个摄像机中心的平面$\pi$上的任意点在第一个相机坐标系下的坐标)经过投影后在两个摄像机成像平面上的投影点分别是x与x'(像素坐标)，平面方程如下：

$$ n^{T} X　+ d = 0 $$

- 继续引入相机成像的几何公式(在第一个相机坐标系下)，有：

$$ Z_1 x = K_1 X $$

$$ Z_2　x' = K_2(R X + t) $$

- 

- 此时存在一个单应矩阵 $H_{\pi}$ 把每一个 $x_i$（$\pi$平面上的任意点$X_i$在第一个相机平面上的投影点）映射到 $x_i'$ ($X_i$在第二个相机平面上的投影点），如下形式：

$$ \left( \begin{array}{ccc} u_2\\  v_2\\   1 \end{array} \right) =  \left( \begin{array}{ccc} h_1& h_2 & h_3\\  h_4& h_5& h_6\\   h_7 & h_8& h_9 \end{array} \right) \left( \begin{array}{ccc} u_1\\  v_1\\   1 \end{array} \right) $$

# 参考资料
- [1] https://www.cnblogs.com/wangguchangqing/p/8287585.html
- [2] https://blog.csdn.net/czl389/article/details/67689625
- [3] https://www.cnblogs.com/wangguchangqing/p/8287585.html
