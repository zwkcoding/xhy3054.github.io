---
layout: post
title: 非线性最小二乘优化方法总结
date: 2019-04-29 10:07:24.000000000 +09:00
img:  sword/sword13.jpg # Add image post (optional)
tag: [数学]
---

# 什么是非线性优化
> 对于最优化问题，如果目标函数是非线性的，那就是非线性优化问题，如果目标函数是凸函数，那就是凸优化问题。

首先需要说明的是，最初的问题其实是一个**最优化问题**。所谓的最优化问题其实就是寻找最优解的问题，一般形式是寻找函数的最大值或者最小值(一般都是将问题转换成求解目标函数的最小值)。

而**非线性优化问题就是针对一个非线性函数求最值的问题**。其实，在我们初高中的时候就已经求解过非线性优化的问题了，比如对于如下非线性函数:

$$ f(x) = x^2 + 2x + 1 $$

求解上面这个非线性函数的最小值，我们可以很轻松的解决，因为这时一个及其简单的非线性优化问题，我们可以直接通过导数为0求解出最小值。**但是对于机器学习中的很多非线性目标函数，我们很难像这样直接求解**。

## 非线性最小二乘问题
首先，什么是**非线性最小二乘问题**，一般它的形式如下：

$$ \min_{x} \frac{1}{2} {\mid \mid f(x) \mid \mid}^2_{2}$$

此处自变量$x \in R^{n} $，f是任意的非线性函数，我们可以假设它有m维：$f(x) \in R^{m}$。

对于不方便直接求解（令导数为0求得所有候选点然后比较）的最小二乘问题，此时一般使用如下迭代的方法求解最小二乘问题。

1. 给定某个初始值$x_{0}$

2. 对于第k次迭代，寻找一个增量${\Delta}x_{k}$，使得${\mid \mid f(x_{k} + {\Delta}x_{k}) \mid \mid}^2_{2}$达到极小值

3. 若${\Delta}x_{k}$足够小，则停止

4. 否则，另$x_{k+1} = x_{k} + {\Delta}x_{k}$，返回到第2步

上述迭代中很重要的一点是增量$\Delta x_{k}$如何确定，下面会开始讲解一些代表性的方法。首先将目标函数在x附近进行泰勒展开：

$$ {\mid \mid f(x + \Delta x ) \mid \mid}^2_{2} \approx {\mid \mid f(x) \mid \mid}^2_{2} + J(x) \Delta x + \dfrac{1}{2} \Delta x^T H \Delta x $$

上面J是函数${\mid \mid f(x) \mid \mid}^2$关于x的**雅可比(Jacobian)矩阵**，而H是其**海塞(Hessian)矩阵**。

> 对于**输入与输出都是向量的函数**，雅可比矩阵是这个函数的一阶偏导数矩阵，海塞矩阵是二阶偏导数矩阵，详细可自行查阅相关资料，此处仅简短说明。

## 最速下降法（梯度下降法，一阶）
**梯度下降法**是只考虑一阶导数的优化方法，优化思想是使用当前位置负梯度方向作为搜索方向，因为该方向为当前位置的最快下降方向，所以也叫做最速下降法。（没什么计算，单纯想法是向负梯度方向前进）

$$ \Delta x = - \lambda J $$

缺点：
1. 靠近极小值时收敛速度减慢
2. 直线搜索时会产生很多问题
3. 可能会出现“之”字形的下降

### 梯度下降法的改进
由于梯度下降法在接近最优解的区域收敛速度明显变慢，因此会需要很多次的迭代，因此出现了一些改进的方法
1. 批量梯度下降法(Batch Gradient Descent, BGD)
2. 随机梯度下降法(Stochastic Gradient Descent, SGD)等

## 牛顿法
**牛顿法**同时考虑一阶与二阶梯度信息，增量方程为

$$ \Delta x = argmin [{\mid \mid f(x) \mid \mid}^2_{2} + J(x) \Delta x + \dfrac{1}{2} \Delta x^T H \Delta x ] $$

上式中右侧等式关于$\Delta x$求导并另它为零，就得到了增量的解。

$$ H \Delta x = -J^T $$

> 牛顿法很好，但是需要计算目标函数的海塞矩阵，这在问题规模较大时十分困难。因此出现了下面两种实用的改进方法

## 高斯牛顿法
高斯牛顿法的做法是将$f(x)$(注意，上面都是${\mid \mid f(x) \mid \mid}^2$)进行泰勒展开：

$$ f(x + \Delta x) \approx f(x) + J(x) \Delta x $$

此处的$J(x)$是函数$f(x)$关于x的雅可比矩阵。因为目标是寻找一个下降矢量$\Delta x$使得${\mid \mid f(x + \Delta x) \mid \mid}^2$达到最小。为了求$\Delta x$，我们会求解如下的最小二乘问题

$$ \Delta x^{\ast} = argmin \dfrac{1}{2} {\mid \mid f(x) + J(x) \Delta x \mid \mid }^2  $$

对上式右侧对$\Delta x$进行求导并另其为0，会得到如下方程组

$$ {J(x)}^T J(x) \Delta x = - {J(x)}^T f(x) $$

上面这个等式是一个关于$\Delta x$的线性方程组，我们称它为**增量方程**，也叫**高斯牛顿方程**或者**正规方程**。我们可以将左侧系数定义为$H$，右侧系数定义为$g$，上式变为

$$ H \Delta x = g $$

> 与牛顿法对比，高斯牛顿法用$J^T J$作为牛顿法中海塞矩阵的近似，从而省略了H的计算。整个**优化问题的核心变成求解增量方程**。

**缺点**：
1. 由于优化过程中我们需要求解增量方程，所以一般这要求H矩阵是可逆的（而且正定的），但实际数据中计算得到的$J^T J$却只有半正定性。也就是说，在使用高斯牛顿法时，可能出现$J^T J$为奇异矩阵或者病态的情况，此时增量稳定性较差，导致算法不收敛。
2. 即使H非奇异也非病态，如果我们求出的步长$\Delta x$太大，也会导致我们采用的局部近似不够准确，甚至我们都没法保证它迭代收敛。

> 对了，高斯牛顿法只能处理二次函数，使用时必须将目标函数转化为二次的。

## 列文伯格-马夸尔特方法
> 由于高斯牛顿法中采用的近似二阶泰勒展开只能在展开点附近有较好的近似效果，所以我们很自然的想到给$\Delta$添加一个**信赖区域**，使得其不会过大。

确定信赖区域的方法：根据我们的近似模型和实际函数之间的差异来确定信赖区域，如果差异小，我们就让范围尽可能大；如果差异大，我们就缩小这个近似范围。使用如下参数

$$ \rho = \dfrac{f(x + \Delta x) - f(x)}{J(x) \Delta x} $$

来判断泰勒近似是否够好，分子是实际函数下降的值，分母是近似模型下降的值。

1. 给定初始值$x_{0}$，以及初始优化半径$\mu$
2. 对于第k次迭代，求解 $\Delta x_{k} = argmin \dfrac{1}{2} {\mid \mid f(x_{k}) + J(x_{k}) \Delta x_{k} \mid \mid}^2 , s.t. {\mid \mid D \Delta x_{k} \mid \mid}^2 \le \mu$，此处$\mu$是信赖区域的半径，D在后面会说明
3. 计算$\rho$
4. 若$\rho > \dfrac{3}{4}$，则$\mu = 2 \mu$
5. 若$\rho < \dfrac{1}{4}$，则$\mu = 0.5 \mu$
6. 如果$\rho$大于某阈值，则认为近似可行。另$x_{k+1} = x_{k} + \Delta x_{k}$
7. 判断算法是否收敛。如果不收敛则返回第2步，否则结束。

上面近似范围扩大的倍数与阈值都是经验值，可以替换成别的数值。在第二步$\Delta x$的求解中，我们将增量限定在一个半径为$\mu$的球里，带上D之后，这个球可以看成一个椭球。通常D为单位矩阵时，是一个正球体。

现在我们来确定LM方法中的$\Delta x$，首先使用拉格朗日乘子法将上述带不等式约束的优化问题转换成一个无约束的优化问题

$$ \Delta x_{k} = argmin \dfrac{1}{2} {\mid \mid f(x_{k}) + J(x_{k}) \Delta x_{k} \mid \mid}^2 + \dfrac{\lambda}{2} {\mid \mid D \Delta x \mid \mid}^2  $$

此处的$\lambda$为拉格朗日乘子。上述做法类似于高斯牛顿法中的做法，对上述问题求解，我们得到下式：

$$ (H + \lambda D_{T}D)\Delta x = g  $$

由上我们发现该问题的核心仍然是计算增量的线性方程，相比较高斯牛顿法的方程，LM中多了一项$\lambda D_{T}D$。考虑上式的简化版本，我们另$D=I$，这个方程变为：

$$ (H + \lambda I)\Delta x = g $$

我们发现，**当$\lambda$比较小时，LM方法近似与高斯牛顿法；当$\lambda$比较大时，说明附近的二次近似不太好，LM方法近似于最速下降法。**

> 列文伯格-马夸尔特方法可在一定程度上米面线性方程组的系数矩阵的非奇异和病态问题，提供更稳定、准确的增量$\Delta x$。尽管它的收敛速度可能会比高斯牛顿法更慢，也被叫做阻尼牛顿法。

## LM算法的另一种理解方式
忽略上面对LM算法来源的说明，其实LM算法主要核心方程是：

$$
\left(\mathbf{J}^{\top} \mathbf{J}+\mu \mathbf{I}\right) \Delta \mathbf{x}_{\mathrm{lm}}=-\mathbf{J}^{\top} \mathbf{f} \quad \text { with } \quad \mu \geq 0
$$

上面这个式子的求解结果为（其中$\mathbf{F}^{\prime \top}$即为$\mathbf{J}^{\top} \mathbf{f}$）：

$$
\Delta \mathbf{x}_{\mathrm{lm}}=-\sum_{j=1}^{n} \frac{\mathbf{v}_{j}^{\top} \mathbf{F}^{\prime \top}}{\lambda_{j}+\mu} \mathbf{v}_{j}
$$

其中，$\lambda_j$ 是 $\mathbf{J}^{\top} \mathbf{J}$ 的第j个特征值，$\mathbf{v}_{j}$是其对应的特征向量。（这个式子可以自行推导）

上面阻尼因子$\mu$作用如下：
- $\mu$大于0保证了$\left(\mathbf{J}^{\top} \mathbf{J}+\mu \mathbf{I}\right)$正定，迭代朝着下降方向进行。

- $\mu$非常大，说明此时高斯牛顿的一次泰勒展开近似效果不好，更新方式偏向近似最速下降法。

- $\mu$比较小，说明此时高斯牛顿的一次泰勒展开近似效果挺好的，更新方式偏向近似高斯牛顿法。

这种方法的核心问题其实是阻尼因子$\mu$的更新策略，更新一般是根据上面提到的$\rho$来进行的。首先，

$$
\begin{aligned} F(\mathbf{x}+\Delta \mathbf{x}) \approx L(\Delta \mathbf{x}) & \equiv \frac{1}{2} (\mathbf{f}(\mathbf{x}) + \mathbf{J} \Delta \mathbf{x})^2 \\ &=\frac{1}{2} \mathbf{f}^{\top} \mathbf{f}+\Delta \mathbf{x}^{\top} \mathbf{J}^{\top} \mathbf{f}+\frac{1}{2} \Delta \mathbf{x}^{\top} \mathbf{J}^{\top} \mathbf{J} \Delta \mathbf{x} \\ &=F(\mathbf{x})+\Delta \mathbf{x}^{\top} \mathbf{J}^{\top} \mathbf{f}+\frac{1}{2} \Delta \mathbf{x}^{\top} \mathbf{J}^{\top} \mathbf{J} \Delta \mathbf{x} \end{aligned}
$$

其次有

$$
\left(\mathbf{J}^{\top} \mathbf{J}+\mu \mathbf{I}\right) \Delta \mathbf{x}_{\mathrm{lm}}=-\mathbf{J}^{\top} \mathbf{f} \quad \text { with } \quad \mu \geq 0
$$

而$\rho$为（分母为更新前后目标函数的变化，分子为近似中应该变化的大小）：

$$
\rho=\frac{F(\mathbf{x})-F\left(\mathbf{x}+\Delta \mathbf{x}_{\operatorname{lm}}\right)}{L(\mathbf{0})-L\left(\Delta \mathbf{x}_{\mathrm{lm}}\right)} = \frac{F(\mathbf{x})-F\left(\mathbf{x}+\Delta \mathbf{x}_{\operatorname{lm}}\right)}{\frac{1}{2} \Delta \mathbf{x}_{\operatorname{lm}}^{\top}\left(\mu \Delta \mathbf{x}_{1 \mathrm{m}}+\mathbf{b}\right)}
$$

在上面已经提到了一种阻尼因子更新的策略（1963年由Marquardt提出）：

$$
\begin{aligned} \text { if } \rho &<0.25 \\ \mu & :=\mu * 2 \\ \text { elseif } \rho &>0.75 \\ \mu & :=\mu / 3 \end{aligned}
$$

但是这个策略不是很好，会出现阻尼因子来回波动的情况，目前在ceres与g2o等优化库中一般采用Nielsen策略：

$$
\begin{array}{l}{\text { if } \rho>0} \\ {\qquad \mu :=\mu * \max \left\{\frac{1}{3}, 1-(2 \rho-1)^{3}\right\} ; \quad \nu :=2} \\ {\text { else }} \\ {\qquad \mu :=\mu * \nu ; \quad \nu :=2 * \nu}\end{array}
$$

在设置阻尼因子$\mu$的初始值时，一般采取的策略是（最大的那个特征值乘以一个常数）：

$$
\mu_{0}=\tau \cdot \max \left\{\left(\mathbf{J}^{\top} \mathbf{J}\right)_{i i}\right\}
$$

其中可以按需设置$\tau \sim\left[10^{-8}, 1\right]$

## Dogleg方法
这种方法与LM方法思想类似，它是高斯牛顿与梯度下降法的混合。

它的做法是，在信任区域内部，使用高斯牛顿法；否则，使用最速下降法。如果最速下降方向在有效范围内部，使用高斯牛顿与梯度下降的混合，在外部，缩小到Region的边界。

## 鲁棒核函数
由于最小二乘法会将残差进行平方，因此如果出现outlier，此时它的误差也会进行平方，如果这个误差很大，会很影响优化的进行，所以通常会在残差上加上一个**鲁棒核函数**来最小化outlier的影响。作用形式如下：

$$
\min _{\mathbf{x}} \frac{1}{2} \sum_{k} \rho\left(\left\|f_{k}(\mathbf{x})\right\|^{2}\right)
$$

将误差的平方项记作$s_{k}=\left\|f_{k}(\mathbf{x})\right\|^{2}$，则鲁棒核误差函数进行二阶泰勒展开有：

$$
\frac{1}{2} \rho(s)=\frac{1}{2}\left(\text { const }+\rho^{\prime} \Delta s+\frac{1}{2} \rho^{\prime \prime} \Delta^{2} s\right)
$$

上述函数中的$\Delta s_{k}$的计算稍微复杂一点为：

$$
\begin{aligned} \Delta s_{k} &=\left\|f_{k}(\mathbf{x}+\Delta \mathbf{x})\right\|^{2}-\left\|f_{k}(\mathbf{x})\right\|^{2} \\ & \approx\left\|f_{k}+\mathbf{J}_{k} \Delta \mathbf{x}\right\|^{2}-\left\|f_{k}(\mathbf{x})\right\|^{2} \\ &=2 f_{k}^{\top} \mathbf{J}_{k} \Delta \mathbf{x}+(\Delta \mathbf{x})^{\top} \mathbf{J}_{k}^{\top} \mathbf{J}_{k} \Delta \mathbf{x} \end{aligned}
$$

将上面两式合并，最终可以近似得到下解：

$$
\begin{aligned} \sum_{k} \mathbf{J}_{k}^{\top}\left(\rho^{\prime} I+2 \rho^{\prime \prime} f_{k} f_{k}^{\top}\right) \mathbf{J}_{k} \Delta \mathbf{x} &=-\sum_{k} \rho^{\prime} \mathbf{J}_{k}^{\top} f_{k} \\ \sum_{k} \mathbf{J}_{k}^{\top} W \mathbf{J}_{k} \Delta \mathbf{x} &=-\sum_{k} \rho^{\prime} \mathbf{J}_{k}^{\top} f_{k} \end{aligned}
$$

### 柯西鲁棒核函数
柯西鲁棒核函数定义为：

$$
\rho(s)=c^{2} \log \left(1+\frac{s}{c^{2}}\right)
$$

其中c为控制参数。对s的一阶导和二阶导为：

$$
\rho^{\prime}(s)=\frac{1}{1+\frac{s}{c^{2}}}, \quad \rho^{\prime \prime}(s)=-\frac{1}{c^{2}}\left(\rho^{\prime}(s)\right)^{2}
$$

## 非线性优化方法存在的问题
1. 首先，对目标函数进行非线性优化有一个很大的前提，那就是**目标函数连续可导**。所以如果目标函数不连续或者不可导，我们就需要想办法使其变得连续可导。比如在slam中处理姿态估计的问题，当姿态使用旋转矩阵与平移变量表示时，显然是不可导的。此时由于欧式矩阵群是一个李群，所以我们会将目标函数映射到其李代数上，此时目标函数就会连续可导了。

2. **初值敏感，并且非线性优化方法求得的只是局部最优解**，这个是由梯度下降的原理决定的，由于初始值不确定，使得很容易陷入局部最优中。一般**凸优化**可以在一定程度上解决这个问题。因为凸函数的局部最优就是全局最优，所以我们可以尝试着将目标函数转换成凸函数，然后再进行优化。
 
> 本文中只是大概讲解了一个非线性优化的整体概念，并针对求解非线性最小二乘问题列举了几种优化方法，还有一些其他的优化方法也具有不错的效果，后面会慢慢补充。。。

## 实践
1. [使用ceres进行函数拟合](https://github.com/xhy3054/myslam/tree/master/03-optimization/ceres_curve_fitting)

2. [使用g2o进行函数拟合](https://github.com/xhy3054/myslam/tree/master/03-optimization/g20_curve_fitting)

3. [只基于Eigen的手写优化算法](https://github.com/xhy3054/vio/tree/master/ch03/CurveFitting_LM)

# 参考文献
- [1] 视觉slam十四讲
