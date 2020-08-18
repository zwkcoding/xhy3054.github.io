---
layout: post
title: pytorch中组成网络的卷积、归一化、relu等常见结构
date: 2020-08-15 10:07:24.000000000 +09:00
img:  one-piece/one-piece27.jpg # Add image post (optional)
tag: [深度学习]
---

对pytorch中组成网络的各种结构进行介绍记录，防止忘掉。

- [第一类-提取特征并调整网络结构的层](#提取并调整特征维度)
- [第二类-归一化调整数值范围层](#归一化调整数值范围层)
- [第三类-非线性激活函数层](#非线性激活函数层)

# 提取并调整特征维度

## 2维卷积层`torch.nn.Conv2d`

> 卷积操作主要用来提取图像特征，卷积过程可以参考我之前的一篇[文章](https://xhy3054.github.io/cnn-base/)中的一个动图。

```py
CLASS torch.nn.Conv2d(	in_channels: int, 
				out_channels: int, 
				kernel_size: Union[int, 
				Tuple[int, int]], 
				stride: Union[int, Tuple[int, int]] = 1, 
				padding: Union[int, Tuple[int, int]] = 0, 
				dilation: Union[int, Tuple[int, int]] = 1, 
				groups: int = 1, 
				bias: bool = True, 
				padding_mode: str = 'zeros')
```

- `in_channels`：一个整型，表示输入通道数；
- `out_channels`：一个整型，表示输出通道数；
- `kernel_size`：卷积核尺寸，可以是一个整型数，也可以是一个包含两个整型的tuple，一个整数时默认长宽相同；
- `stride`： 输入可以是一个整型号，也可以是一个包含两个整型的tuple；表示步长；一般步长也可以看作提取到的特征相对于原图尺寸缩小的倍数；
- `padding`：输入可以是一个整型号，也可以是一个包含两个整型的tuple；表示默认在图像边缘补0的尺寸，输入一个tuple的话，第一个数表示高度上，第二个表示宽度上填充的尺寸；比如当为1时，表示原图基础上，上下左右各补了一行；
- `dilation`：输入可以是一个整型号，也可以是一个包含两个整型的tuple；表示卷积核的扩张尺寸，也是核点之间的间距；
- `groups`: 范围从1到`in_channels`，默认是1，表示所有每个输出通道都包含了所有输入通道的信息；如果为2，则一半的输出通道是由一半的输入通道产生的，另一半的输出通道是由剩下一半的输入通道产生；如果为`in_channels`，表示每个输入通道都有自己单独的卷积核，卷积核输出维度为`out_channels/in_channels`；

## 上采样层`torch.nn.PixelShuffle`

> 主要用来重新排列一个维度(x, C*r^2, H, W)的tensor为一个维度(x, C, H*r, W*r)的tensor。这种做法是16年一篇论文提出的超分辨率方法，主要思想是对于（1, H, W）维度的图像，为了将其放大r倍，首先使用卷积网络提取得到(r^2, H, W)维度的深层特征；然后使用PixelShuffle对深层特征重新排列，得到一个(1, H*r, W*r)的高分辨的图像。

```python
CLASS torch.nn.PixelShuffle(upscale_factor: int)
```

- `upscale_factor`：int类型，上采样的倍数，也是上面的r；


# 归一化调整数值范围层

## 批处理归一化层`torch.nn.BatchNorm2d`

> 批处理归一化层主要用来进行数据归一化，使得在relu之前不会因数据过大而导致网络性能不稳定。对于一个（1,3，H,W）的输入，主要的处理如下：

$$y=\frac{x-\mathrm{E}[x]}{\sqrt{\operatorname{Var}[x]+\epsilon}} * \gamma+\beta$$

上面计算是对3个通道分别进行的，其中x为一个一个像素位置多个通道值组成的向量，E为每个通道的平均数组成的向量，var为每个通道的标准差组成的向量，$\epsilon$是为了数值稳定性在分母上增加的值，$\gamma$（默认1）和$beta$（默认0）为可学习的参数向量（维度为通道数）。上面公式写成向量形式，但是实际理解可以是对特征的每个通道分别执行上式。


```py
CLASS torch.nn.BatchNorm2d(num_features, 
	eps=1e-05, 
	momentum=0.1, 
	affine=True, 
	track_running_stats=True)
```

- `num_features`：当前要处理的特征层的通道数（一开始rgb图像的话是3）；
- `eps`：也就是上面公式中的$\epsilon$，为数值稳定性在分母上增加的值。默认为：1e-5；
- `momentum`：一个用于运行过程中均值和方差的一个估计参数（我的理解是一个稳定系数，类似于SGD中的momentum的系数）；
- `affine`：当设为true时，该模块拥有可以学习的系数$\gamma$和$beta$；
- `track_running_stats`: 是否跟踪均值和标准差的计算；

## 归一化层`torch.nn.functional.normalize`

> 在指定的维度上执行$L_p$归一化操作，其实就是在某个维度上对每个元素做$L_p$操作并做sum，然后当前元素值为原值除以sum，公式为：

$$v=\frac{v}{\max \left(\|v\|_{p}, \epsilon\right)}$$


```python
torch.nn.functional.normalize(input: torch.Tensor, 
	p: float = 2, 
	dim: int = 1, 
	eps: float = 1e-12, 
	out: Optional[torch.Tensor] = None) → torch.Tensor
```

- `input`：任意shape的tensor输入；
- `p`：范数的指数，默认为2，即$L_2$范数；
- `dim`：计算范数的维度，默认是1；
- `out`：the output tensor. If out is used, this operation won’t be differentiable.

## 层`torch.nn.functional.softplus`

> 逐元素执行如下操作，note:为了数值稳定，当$\beta x > threshold$时实现变为线性函数。

$$\text { Softplus }(x)=\frac{1}{\beta} * \log (1+\exp (\beta * x))$$

```python
torch.nn.functional.softplus(input, beta=1, threshold=20) → Tensor
```

## 层`torch.nn.functional.softmax`

> 也是一种归一化操作，沿着某个维度执行如下操作，使得该维度向量每个元素值在0到1之间，并且和为1。

$$\operatorname{Softmax}\left(x_{i}\right)=\frac{\exp \left(x_{i}\right)}{\sum_{j} \exp \left(x_{j}\right)}$$

```python
torch.nn.functional.softmax(input: torch.Tensor, 
	dim: Optional[int] = None, 
	_stacklevel: int = 3, 
	dtype: Optional[int] = None) → torch.Tensor
```

- `input`：输入tensor；
- `dim`：沿着这个维度执行softmax
- `dtype`：如果指定类型，在执行操作前需要将输入被转换为该类型；

# 非线性激活函数层

## 修正线性单元层`torch.nn.ReLU`

> 主要是执行非线性变换。Relu的非线性变换是大于0时不变，小于0时置为0，特点是瘦脸快，求梯度简单。

```python
CLASStorch.nn.ReLU(inplace: bool = False)
```

- `inplace`：是否在原数据上操作；

## 第二种激活函数`torch.nn.ELU`

> 对每个元素执行如下操作：

$$ ELU(x)=max(0,x)+min(0,α∗(exp(x)−1)) $$

```python
CLASStorch.nn.ELU(alpha: float = 1.0, inplace: bool = False)
```

- `alpha`：上面公式中的$\alpha$值，默认为1；
- `inplace`：是否在原数据位置进行操作；


# 参考文献

- [1] pytorch官方文档

