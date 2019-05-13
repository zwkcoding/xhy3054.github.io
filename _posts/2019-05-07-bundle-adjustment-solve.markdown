---
layout: post
title: Bundle Adjustment在视觉slam中的应用与求解
date: 2019-05-07 10:07:24.000000000 +09:00
img:  one-piece/one-piece1.jpg # Add image post (optional)
tag: [计算机视觉]
---
在视觉slam与sfm中，BA是很常见的参数优化方法。它的全名是 Bundle Adjustment，中文翻译有捆集调整、光束法平差等。

> BA的主要思想是**最小化重投影误差**，以其作为目标函数来优化位姿、地图点、相机参数等。

# 计算投影点
根据以前的两篇博客[相机成像模型](https://xhy3054.github.io/camerca-module/)与[相机畸变与标定](https://xhy3054.github.io/camera-calibration-undistort/)，我们可以知道将一个地图点投影到像素平面上流程如下：

- 首先，世界坐标系下有一个固定的点P，**世界坐标**是 $P_{w}$

- 获得点P的**相机坐标系坐标**，可以利用相机外参R与t得到 $ P_{c} = RP_{w} + t $

- 在2中获得的 $P_c$ 仍然是三维的 (X,Y,Z) ，将其投影到归一化平面 $Z=1$ 上，得到**归一化相机坐标**：$ P_{C} = [X/Z, Y/Z, 1]^T $

- 此处加入**畸变**因素，畸变公式如下(其中，$x=X/Z, y=Y/Z$)：

$$ x_{distorted} = x( 1 + k_{1} r^2 + k_{2} r^4 + k_{3} r^6) + [ 2p_{1}xy + p_{2}(r^2+2x^2)]  \\ y_{distorted} = y( 1 + k_{1} r^2 + k_{2} r^4 + k_{3} r^6) + [ p_{1}(r^2+ 2y^2)+ 2p_{2}xy]   $$


- 最后一步，利用内参K，将归一化相机坐标转换成**像素坐标**： $ P_{uv} = KP_{distorted} $

$$ u = f_{x} x_{distorted} + c_{x} \\ v = f_{y} y_{distorted} + c_{y}  $$

> 上述过程就是前面[状态估计之非线性最小二乘](https://xhy3054.github.io/state-estimation-nonlinear-least-square/)中提到的**观测方程**。之前我们抽象的将其记作$z = h(x, y)$。

# BA的目标函数
在BA中，待优化的变量有相机的位姿$x$，即外参$R,t$，它对应的李代数为$\xi$；地图点y的三维坐标$p$。而观测数据是像素坐标$\boldsymbol{z} \triangleq\left[u, v\right]^{T}$。所以每次观测方程的误差是

$$
e=z-h(\xi, p)
$$

我们将不同时刻对不同地图点的观测都考虑进来，比如$$\boldsymbol{z}_{i j}$$为在位姿$$\boldsymbol{\xi}_{i}$$处观察地图点$$\boldsymbol{p}_{j}$$产生的数据，那么整体的**目标函数**为：

$$
\frac{1}{2} \sum_{i=1}^{m} \sum_{j=1}^{n}\left\|\boldsymbol{e}_{i j}\right\|^{2}=\frac{1}{2} \sum_{i=1}^{m} \sum_{j=1}^{n}\left\|\boldsymbol{z}_{i j}-h\left(\boldsymbol{\xi}_{i}, \boldsymbol{p}_{j}\right)\right\|^{2}
$$

> BA的任务就是求解这个最小二乘问题。

# BA的求解
由上面我们知道我们需要求解的是一个非线性优化问题，很自然的我们想到高斯牛顿等方法。使用这些方法，我们首先将所有待优化的变量组织成一个向量$x$，这个向量便是目标函数的自变量：

$$
\boldsymbol{x}=\left[\boldsymbol{\xi}_{1}, \ldots, \boldsymbol{\xi}_{m}, \boldsymbol{p}_{1}, \ldots, \boldsymbol{p}_{n}\right]^{T}
$$

我们每次迭代时都需要寻找一个$$\Delta \boldsymbol{x}$$，它是自变量的迭代增量。利用高斯牛顿的原理，我们为自变量添加一个增量时，目标函数变为：

$$
\frac{1}{2}\|f(\boldsymbol{x}+\Delta \boldsymbol{x})\|^{2} \approx \frac{1}{2} \sum_{i=1}^{m} \sum_{j=1}^{n}\left\|\boldsymbol{e}_{i j}+\boldsymbol{F}_{i j} \Delta \boldsymbol{\xi}_{i}+\boldsymbol{E}_{i j} \Delta \boldsymbol{p}_{j}\right\|^{2}
$$

其中$$\boldsymbol{F}_{i j}$$表示j地图点投影到i时刻相机像素平面投影残差对i时刻相机姿态的偏导数，而$\boldsymbol{E}_{i,j}$表示j地图点投影到i时刻相机像素平面上的投影残差对j地图点的三维世界坐标的偏导数。

我们将相机位姿变量放在一起：

$$
\boldsymbol{x}_{c}=\left[\boldsymbol{\xi}_{1}, \boldsymbol{\xi}_{2}, \ldots, \boldsymbol{\xi}_{m}\right]^{T} \in \mathbb{R}^{6 m}
$$

将地图点三维坐标变量放在一起：

$$
\boldsymbol{x}_{p}=\left[\boldsymbol{p}_{1}, \boldsymbol{p}_{2}, \ldots, \boldsymbol{p}_{n}\right]^{T} \in \mathbb{R}^{3 n}
$$

然后使用大矩阵表示矩阵简化上面公式

$$
\frac{1}{2}\|f(\boldsymbol{x}+\Delta \boldsymbol{x})\|^{2}=\frac{1}{2}\left\|\boldsymbol{e}+\boldsymbol{F} \Delta \boldsymbol{x}_{c}+\boldsymbol{E} \Delta \boldsymbol{x}_{p}\right\|^{2}
$$

此处我们将原本由很多小型二次项之和的形式变成了一个更整体的样子。此处$E$与$F$是整体目标函数对整体变量的导数，他们是比较大的偏导数矩阵，其中有很多项是0。

## 使用高斯牛顿进行求解
上面已经将投影误差进行了泰勒一阶展开，此时不论是使用高斯牛顿还是LM方法对其进行优化，都需要求解一个方程

$$
\boldsymbol{H} \Delta \boldsymbol{x}=\boldsymbol{g}
$$

如果使用$GN$法，则$H$取$\boldsymbol{J}^{T} \boldsymbol{J}$；如果使用$LM$法，则$H$取$\boldsymbol{J}^{T} \boldsymbol{J}+\lambda \boldsymbol{I}$。不管哪种方法，其中$J$都是

$$
J=[\begin{array}{cc} \boldsymbol{F} & \boldsymbol{E} \end{array}]
$$

如果是高斯牛顿，则$H$矩阵为

$$
\boldsymbol{H}=\boldsymbol{J}^{T} \boldsymbol{J}=\left[ \begin{array}{cc}{\boldsymbol{F}^{T} \boldsymbol{F}} & {\boldsymbol{F}^{T} \boldsymbol{E}} \\ {\boldsymbol{E}^{T} \boldsymbol{F}} & {\boldsymbol{E}^{T} \boldsymbol{E}}\end{array}\right]
$$

> 其实无论是$GN$还是$LM$方法，我们都需要求上述的$H$矩阵。不过这个矩阵的维度非常大，直接计算很耗时。

### H矩阵的稀疏性与边缘化
> **H矩阵的稀疏性是由该问题的图优化模型的固有属性决定的**。

如下一个图模型，由两个相机位姿节点与六个地图坐标节点组成，总共有八条边。
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/BA/graph.PNG"   width="750" height="350"/>
</div>

对于上述图模型，我们构建目标函数，并求解其$J$与$H$矩阵，会得到如下形式的结果。
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/BA/jacobian.PNG"   width="750" height="350"/>
</div>

**对于一条边（一个误差项），只对其关联的节点的偏导数不为0。**图中$H$矩阵的行列维度都是节点的个数，前两行（与前两列）是相机姿态节点的关联关系，后六行（与后六列）是地图点坐标节点的关联关系。图中$J$矩阵的行数是图中边的个数，列数是图中节点的个数，每一行是对每一个误差项（边）求的偏导数向量，每一列可以理解为对应的节点都关联到了哪条边上。

<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/BA/link.PNG"   width="750" height="350"/>
</div>

上图中的H矩阵除了对角线上的元素外与图的**邻接矩阵**具有完全一致的结构。由上，我们可以讲H矩阵划分成如下四个部分组成。
<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/BA/Hessa.PNG"   width="500" height="400"/>
</div>

> 注意，这个矩阵的独特性质是，**左上角的矩阵$B$与右下角的矩阵$C$是两个对角矩阵，E矩阵如何由图的性质决定。**这种稀疏性是由这种图模型的连接属性决定的，这种图模型有一个特点，**那就是每个相机位姿节点只与地图点节点关联，与其他相机节点不关联（因为没考虑运动方程）这使得$B$是对角矩阵，而地图点节点只与相机节点关联，与其他地图点不关联，这使得$C$是对角矩阵。**

这样，对应的增量方程$H \Delta x=g$也就变为

$$
\left[ \begin{array}{cc}{\boldsymbol{B}} & {\boldsymbol{E}} \\ {\boldsymbol{E}^{T}} & {\boldsymbol{C}}\end{array}\right] \left[ \begin{array}{c}{\Delta \boldsymbol{x}_{c}} \\ {\Delta \boldsymbol{x}_{p}}\end{array}\right]=\left[ \begin{array}{c}{\boldsymbol{v}} \\ {\boldsymbol{w}}\end{array}\right]
$$

我们可以**利用$H$的稀疏性来加速求解这个方程**。有很多方法，在slam中最常用的是：$Schur$消元，也叫作$Marginalization$（边缘化）

B的维度是相机位姿节点的个数，C的维度是地图点的个数，通常前者远远小于后者。然后对角矩阵求逆的难度远远小于普通矩阵求逆，我们可以对上述方程组进行高斯消元，消去右上角的非对角部分$E$：

$$
\left[ \begin{array}{cc}{\boldsymbol{I}} & {-\boldsymbol{E} \boldsymbol{C}^{-1}} \\ {\mathbf{0}} & {\boldsymbol{I}}\end{array}\right] \left[ \begin{array}{cc}{\boldsymbol{B}} & {\boldsymbol{E}} \\ {\boldsymbol{E}^{T}} & {\boldsymbol{C}}\end{array}\right] \left[ \begin{array}{c}{\Delta \boldsymbol{x}_{c}} \\ {\Delta \boldsymbol{x}_{p}}\end{array}\right]=\left[ \begin{array}{cc}{\boldsymbol{I}} & {-\boldsymbol{E} C^{-1}} \\ {\mathbf{0}} & {\boldsymbol{I}}\end{array}\right] \left[ \begin{array}{c}{\boldsymbol{v}} \\ {\boldsymbol{w}}\end{array}\right]
$$

整理可得

$$
\left[ \begin{array}{cc}{\boldsymbol{B}-\boldsymbol{E} \boldsymbol{C}^{-1} \boldsymbol{E}^{T}} & {\mathbf{0}} \\ {\boldsymbol{E}^{T}} & {\boldsymbol{C}}\end{array}\right] \left[ \begin{array}{c}{\Delta \boldsymbol{x}_{c}} \\ {\Delta \boldsymbol{x}_{p}}\end{array}\right]=\left[ \begin{array}{c}{\boldsymbol{v}-\boldsymbol{E} \boldsymbol{C}^{-1} \boldsymbol{w}} \\ {\boldsymbol{w}}\end{array}\right]
$$

这样，第一行方程组就变成与$\Delta x_{p}$无关了，此时我们将其单独拿出来，得到仅关于位姿部分的增量方程

$$
\left[\boldsymbol{B}-\boldsymbol{E} \boldsymbol{C}^{-1} \boldsymbol{E}^{T}\right] \Delta \boldsymbol{x}_{c}=\boldsymbol{v}-\boldsymbol{E} \boldsymbol{C}^{-1} \boldsymbol{w}
$$

这个线性方程组的维度与$B$矩阵一致。求解这个方程的计算量就会远远小于求解原方程了。最后再将这个方程的解带入原方程就可以求解出$\Delta x_{p}$了。**这就是边缘化求解的整个过程**了。

上述方程左边的系数我们记为$S$，它是有其物理意义的。$S$矩阵的非对角线上的非零矩阵块，表示了该处对应的两个相机位姿节点之间存在共同观测的地图点。如下

<div style="text-align: center">
<img src="{{site.baseurl}}/assets/img/BA/schur.PNG"   width="750" height="420"/>
</div>

> 上图只列出了前四个相机位姿节点的共视关系，整个方形矩阵是所有相机位姿节点的共视关系。

### $Schur$消元的优势
1. 在消元过程中，由于$C$为对角块，所以$C^{-1}$容易求解

2. 求解了$\Delta x_{c}$之后，路标部分的增量方程可以通过$$\Delta \boldsymbol{x}_{p}=\boldsymbol{C}^{-1}\left(\boldsymbol{w}-\boldsymbol{E}^{T} \Delta \boldsymbol{x}_{c}\right)$$求得。这也用到了$C^{-1}$易于求解的特性。

3. 我们发现上面的消元过程其实不需要$B$矩阵是对角矩阵，即考虑运动方程也可以。

4. 不一定非要只消元地图点，也可以反过来消元相机位姿节点，选择比较有效的做法即可。

## BA中的核函数
在上面的分析中，我们使用误差项的二范数平方和作为目标函数，这种做法会使得如果出现误匹配之类的情况，错误边对整体优化的影响会很大，因为错误边的误差的二范数平方增长太快了。

此时我们可以使用一些核函数来代替二范数平方的操作，使得误差的增长不是那么快，同时可以保证自己的光滑性质。我们也叫它们**鲁棒核函数**。

一种常见的核函数是$Huber$核：

$$
H(e)=\left\{\begin{array}{ll}{\frac{1}{2} e^{2}} & {\text { if }|e| \leq \delta} \\ {\delta\left(|e|-\frac{1}{2} \delta\right)} & {\text { otherwise }}\end{array}\right.
$$

> 还有一些其他的核函数，比如$Cauchy$核、$Tukey$核等等，在$g2o$与$Ceres$等优化库中都有提供。

## 工程实例
1. [使用g2o求解BA]()

2. [使用ceres求解BA]()

# 拓展
## pose graph的优化
> 通常在slam的后端优化中，在经过几轮的ba迭代后（几轮观测后），地图点的位置便已经近乎固定了，此时如果继续在新的相机帧进来后进行全局的ba优化，会有些浪费计算资源。

此时，我们一般倾向于只优化相机位姿节点，即构建一个只由pose组成的图，并对其进行优化，这就叫做**位姿图(pose graph)的优化**，具体推导过程与ba类似，计算雅克比什么的，可以自行翻阅资料。

## 因子图优化
> **因子图**是基于概率图（主要是贝叶斯网络）的图优化。

因子图是一种无向图，节点有两种，一种是需要优化的**变量节点**（比如slam中的位姿与路标），一种是不需要优化的**因子节点**（比如slam中的观测与输入）。因子图与普通的图优化都是最小二乘问题，并且稀疏性也类似。

因子图比较特殊的地方在于，其可以**增量式**的处理后端优化。比如，在**里程计的因子图优化**中，当新的节点与边被加入到图中时，我们只需要考虑最后一个与之相连的节点，可以近似的认为早先节点的估计值不发生变换（其实还是会有影响的，不过相比于最近的节点而言很小），因此可以节省很多计算量，**每次新加节点时，没必要对整个图进行优化**。

> 需要注意的一点是，如果图中包含回环检测的部分，那么受影响的范围应该是当前回环上的所有节点，也就是这部分回环的轨迹都需要进行调整。

# 参考文献
- [1]. 视觉slam十四讲
