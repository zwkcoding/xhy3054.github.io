---
layout: post
title: 理解齐次坐标
date: 2019-01-11 10:07:24.000000000 +09:00
img:  game1.jpg # Add image post (optional)
tag: [数学]
---
齐次坐标系（Homogeneous Coordinates）是计算机视觉与图形学中的一个重要的数学工具。

# 齐次坐标的引入
在**欧式空间**里，两条公面的平行线无法相交，但是在**投影空间**(Projective Space)里不是这样。一个直观的表示如下：两条轨道的间距随着视线变远而逐渐变小，直到在无限远处相交。
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/homogeneous_coordinates/railway.jpg"/>
</div>
在欧式空间里采用$(x, y, z)$表示一个三维点，但是**无穷远点**$(\infty, \infty, \infty)$在欧式空间里是没有意义的，**在投影空间中进行图形和几何运算**并不是一个简单的问题，为了解决这个问题，数学家　August Ferdinand Möbius　提出了齐次坐标系，使用 N+1 个量来表示　N 维坐标。例如在二维齐次坐标系中，我们引入一个量w，将一个二维点$(x, y)$重新表示为$(X, Y, w)$的形式，其中转换关系为：

$$ x = \frac{X}{w} $$

$$ y = \frac{Y}{w} $$

例如，欧式坐标中的一个二维点$(1, 2)$可以在齐次坐标中表示为$(1, 2, 1)$，如果点逐渐移动向无穷远处，其欧式坐标变为$(\infty, \infty, \infty)$，齐次坐标变为$(1, 2, 0)$。其中齐次坐标在表示无穷远处的点时不需要用到$\infty$。

> 其中齐次坐标$(1, 2, 1)$等价于齐次坐标$(2, 4, 2)$...即$(k, 2k, k)，k \in R$，此处这些点具有尺度不变性，是齐性的，所以称为**齐次坐标**

## 平行线相交的不太严格的证明
欧式空间中假设有如下两条平行线:

$$ Ax + By + C = 0 $$

$$ Ax + By + D = 0 $$

上面两条先在欧式空间中除非$C = D$，否则不相交。使用$\frac{x}{w}, \frac{y}{w}$替换$x, y$(正如前文提到的使用$N+1$个量表示N维坐标，这里增加了一个量w)，可以得到：

$$ Ax + By + Cw = 0 $$

$$ Ax + By + Dw = 0 $$

上式可以得到解$(x, y, 0)$，即两条平行线的齐次坐标表示在$(x, y, 0)$也就是无穷点处相遇

> 当然这只是一个不严格不严谨的表示，齐次坐标真正的作用在于下文。

# 齐次坐标可以区分点与向量
以二维空间为例，$(a, b)$这样的表示既可以是一个坐标表示，也可以是一个向量表示。假设这个坐标系$xOy$中两个基向量为$\vec{x}, \vec{y}$，坐标原点为o，则其中
- 表示向量$\vec{v}$时，代表$\vec{v} = a\vec{x} + b\vec{y}$
- 表示一个点p时，代表$ p - o = a\vec{x} + b\vec{y} $

如果没有附加说明，我们不能区别$(a, b)$表示的是向量还是点。用三个量来表示的化，我们可以明确的区分向量和点
- 齐次点$(a, b, 1)$
- 齐次向量$(a, b, 0)$

# 齐次坐标更易用于仿射变换
一个二维点的仿射变换可以用一个矩阵乘法(线性变换)与一个矩阵加法(平移变换)的叠加来表示：
- 矩阵乘法可以表示线性变换，线性变换可以表示旋转变换或者缩放变换（不局限于表示这两种）

$$ \left[ \begin{array}{ccc} x'\\  y' \end{array} \right] = \left[ \begin{array}{ccc} cos(\theta) & -sin(\theta) \\  sin(\theta) & cos(\theta) \end{array} \right] \left[ \begin{array}{ccc} x\\  y \end{array} \right]$$

$$ \left[ \begin{array}{ccc} x'\\  y' \end{array} \right] = \left[ \begin{array}{ccc} S_x & 0 \\  0 & S_y \end{array} \right] \left[ \begin{array}{ccc} x\\  y \end{array} \right] $$

- 矩阵加法可以表示平移变换（不局限于表示平移）

$$ \left[ \begin{array}{ccc} x'\\  y' \end{array} \right] = \left[ \begin{array}{ccc} x'\\  y' \end{array} \right] + \left[ \begin{array}{ccc} t_x\\  t_y \end{array} \right] $$

如上，我们发现原本的平移变换不能用矩阵相乘的形式表达。在引入齐次坐标后（具有尺度不变性，实际上在高一维的空间映射到$w=1$平面后的结果可直接导出到欧式空间）
- 旋转变换和尺度变换

$$ \left[ \begin{array}{ccc} x'\\  y'\\ 1 \end{array} \right] = \left[ \begin{array}{ccc} cos(\theta) & -sin(\theta) & 0 \\  sin(\theta) & cos(\theta) & 0 \\ 0 & 0 & 1 \end{array} \right] \left[ \begin{array}{ccc} x\\  y \\ 1 \end{array} \right]$$

$$ \left[ \begin{array}{ccc} x'\\  y'\\ 1 \end{array} \right] = \left[ \begin{array}{ccc} S_x & 0 & 0 \\  0 & S_y & 0\\ 0 & 0 & 1\end{array} \right] \left[ \begin{array}{ccc} x\\  y\\ 1 \end{array} \right] $$

- 平移变换

$$ \left[ \begin{array}{ccc} x'\\  y'\\ 1 \end{array} \right] = \left[ \begin{array}{ccc} 1 & 0 & t_x \\  0 & 1 & t_y\\ 0 & 0 & 1\end{array} \right] \left[ \begin{array}{ccc} x\\  y\\ 1 \end{array} \right] $$

- 这样，我们可以导出整个仿射变换的矩阵形式

$$ T = \left[ \begin{array}{ccc} A & t \\ O_{1*2} & 1\end{array} \right] \left[ \begin{array}{ccc} x\\  y\\ 1 \end{array} \right] $$

仿射变换保留了点的共线、共面、比例等关系，是图形处理中十分重要的一环。而齐次坐标的引入**使得仿射变换能够以一种紧凑统一的矩阵形式表示和计算**。

# 总结
> 齐次坐标表示是计算机图形学的重要手段之一，它既能够用来**明确的区分向量和点**，同时也更**易于进行仿射几何变换**。　—— F.S.Hill Jr.《计算机图形学(OpenGL版)》

# 参考资料
- [1] 《计算机图形学(OpenGL版)》-　F.S.Hill Jr.
- [2] 《Computer Vision: Algorithms and Applications》- Richard Szeliski 
- [3]  https://oncemore.wang/blog/homogeneous/

