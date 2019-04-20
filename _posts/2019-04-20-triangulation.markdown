---
layout: post
title: 单目vo中的深度确定方法--三角测量 
date: 2019-04-20 10:07:24.000000000 +09:00
img:  sword/sword8.jpg # Add image post (optional)
tag: [图像处理]
---
三角测量的目的是用来确定图片中某一个点的深度。为什么会有这样的需求呢？我们在前面的博客中提到了[对极几何](https://xhy3054.github.io/epipolar-geometry/)与[单应变换](https://xhy3054.github.io/Homography-matrix/)。在前面其实已经提到过了，我们可以通过本质矩阵与单应矩阵恢复出相机变换的位姿，但是这两种方法确定的位姿变换是具有**尺度不确定性**的。

> 因为基础矩阵与单应矩阵本身描述的是从**一个2d平面到另一个2d平面的变换**，无需3d深度信息，这两种变换都是正常进行的，因此自然就具有尺度不确定性。更加具体的讲，将基础矩阵与单应矩阵作用在一个齐次坐标(x,y,1)上时，此时，随意的将基础矩阵与单应矩阵进行一定尺度缩放，由于**齐次坐标的尺度等价性**，对最终的变换结果都不会产生影响。

## 三角测量
三角测量本身是由高斯提出，最早应用在天文地理领域，根据不同季节观察到的星星的角度，估计星星与我们的距离。

在相机几何中的应用，推导如下：

由[相机成像的几何描述](https://xhy3054.github.io/camerca-module/)我们可以理解如下的公式(世界坐标到像素坐标的转换)：

$$ Z_1p_{uv1} = K_1P_w   $$

$$ Z_2p_{uv2} = K_2(RP_w + t)   $$

其中K是相机内参，R与t是第二个相机在第一个相机的相机坐标系下的外参，$P_w$是此空间点在第一个相机的相机坐标系下的坐标。Z是空间点到相机光心的距离(也是相机坐标系下的z轴坐标)。$p_{uv1}$与$p_{uv2}$是空间点$P_w$在两个相机平面上的投影点。

- 首先做出如下定义（其中$x_1$与$x_2$是归一化的相机坐标（X/Z,Y/Z,1））：

$$ x_1 = K_1^{-1}p_{uv1} $$

$$ x_2 = K_2^{-1}p_{uv2} $$ 

- 带入如上定义可得：

$$ Z_2 x_2 = Z_1 Rx_1 + t $$

- 由于我们已经通过本质矩阵分解或者单应矩阵分解获得了R与t，此时想求的是两个特征点的深度，即上式中的$Z_1$与$Z_2$，此时，可通过如下操作分别求出，首先求解$Z_1$，上式两边同时左乘`x2^`，也就是两侧同时与$x_2$做外积：

$$ Z_2 x_2^{\land} x_2 = 0 = Z_1 x_2^{\land}Rx_1 + x_1^{\land}t $$

如上，可以直接求出$p_{uv1}$的深度$Z_1$了，然后$Z_1$也可以很轻松地求出。

## 代码实现
```cpp
void triangulation ( 
    const vector< KeyPoint >& keypoint_1, 
    const vector< KeyPoint >& keypoint_2, 
    const std::vector< DMatch >& matches,
    const Mat& R, const Mat& t, 
    vector< Point3d >& points )
{
    //相机第一个位置处的位姿
    Mat T1 = (Mat_<float> (3,4) <<
        1,0,0,0,
        0,1,0,0,
        0,0,1,0);
    //相机第二个位置处的位姿
    Mat T2 = (Mat_<float> (3,4) <<
        R.at<double>(0,0), R.at<double>(0,1), R.at<double>(0,2), t.at<double>(0,0),
        R.at<double>(1,0), R.at<double>(1,1), R.at<double>(1,2), t.at<double>(1,0),
        R.at<double>(2,0), R.at<double>(2,1), R.at<double>(2,2), t.at<double>(2,0)
    );
    
    // 相机内参
    Mat K = ( Mat_<double> ( 3,3 ) << 520.9, 0, 325.1, 0, 521.0, 249.7, 0, 0, 1 );
    vector<Point2f> pts_1, pts_2;
    for ( DMatch m:matches )
    {
        // 将像素坐标转换至相机平面坐标，为什么要这一步，上面推导中有讲
        pts_1.push_back ( pixel2cam( keypoint_1[m.queryIdx].pt, K) );
        pts_2.push_back ( pixel2cam( keypoint_2[m.trainIdx].pt, K) );
    }
    
    Mat pts_4d;
    //opencv提供的三角测量函数
    cv::triangulatePoints( T1, T2, pts_1, pts_2, pts_4d );
    
    // 转换成非齐次坐标
    for ( int i=0; i<pts_4d.cols; i++ )
    {
        Mat x = pts_4d.col(i);
        x /= x.at<float>(3,0); // 归一化
        Point3d p (
            x.at<float>(0,0), 
            x.at<float>(1,0), 
            x.at<float>(2,0) 
        );
        points.push_back( p );
    }
}

Point2f pixel2cam ( const Point2d& p, const Mat& K )
{
    return Point2f
    (
        ( p.x - K.at<double>(0,2) ) / K.at<double>(0,0), 
        ( p.y - K.at<double>(1,2) ) / K.at<double>(1,1) 
    );
}
```

# 参考文献
- [1] 《视觉slam十四讲》
