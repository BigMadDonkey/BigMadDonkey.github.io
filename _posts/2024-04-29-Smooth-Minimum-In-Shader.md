---
layout: post
title: 平滑最小值(SmoothMinimum)及在Shader中的应用
categories: [Shader]
description: 本文介绍在Shader中常用的一类函数：平滑最小值(smin)。
keywords: Shader
mathjax: true
---

{:toc}

## 引言

本文介绍在Shader中常用的一类函数：平滑最小值(smooth minimum)的原理及其应用。主要参考了[iq的博客](https://iquilezles.org/articles/smin/ "iq Blog")。


## 平滑最小值的原理

在Shader中，在两个value之间取最小值`min(a, b)`的操作非常常见(例如构建SDF形状的过程），但是这样的函数在相邻的点之间并不是连续变化的，这就会导致形状的突然变化。以一个3D形状为例，如果两个独立的SDF之间有相交区域，不经过平滑的话，就会在它们的相交或边缘接近处看到明显的表面变化，像一张折叠起来的锋利纸片。

![iq的示例](https://iquilezles.org/articles/smin/gfx00.jpg)

而Smooth Minimum的作用就是将求二者中较小值的操作平滑一下，使得**在a、b相差不会过大（未超出一定范围）时，得到二者的一个平滑过后的最小值（通常，这会比二者都小）**。用Smooth Minimum（下文简称smin）来替换min，那么不同shape间将要接触的边缘会有一层比较平滑的膨胀，看起来就会好多了。

![iq的示例2](https://iquilezles.org/articles/smin/gfx01.jpg)

想法不难理解，关键就在于如何实现这样的平滑函数。实际上只要符合这个需求，可以写出无数个用于平滑的函数，但是一般而言常用的smin函数更符合以下的定义：给定常数k > 0，在$abs(b - a) < k$，亦即$(b - a) \in (-k, k)$时，$smin(a, b, k) < min(a, b)$，否则则取$min(a, b)$。其中k代表平滑因子，k的大小就决定了平滑的强度，k越大平滑的容忍区间越大，a - b如果超出了$(-k, k)$的区间就超出了平滑的容忍范围，那么就不做处理了。

> 在实际使用中通常对点的坐标做归一化，那么这时k一定是在$(0,1)$范围内的。

## 常用的smin的写法

在iq的blog中提及了一种对smin函数实现的分类方式：

借助于一个kernel函数$g(x)$,来做到根据不同的平滑因子k，引入在a，b之间的平滑方式。这里要求$\lim\limits_{x \to \infty}g(x) = x, \lim\limits_{x \to -\infty}g(x) = 0, g(x) \in [0, 1] \ while \ x\in(-1, 1)$。其中的x是我们传入的归一化的两个value之差。
这样的smin函数形如

```glsl
float smin( float a, float b, float k )
{
    k /= g(0.0);
    float x = (b-a)/k;
    return b - k*g(x);
}
// or, optimized one
float smin( float a, float b, float k )
{
    k /= g(0.0);
    float h = max(k-abs(a-b),0.0)/k;
    return min(a,b) - k*g(h-1.0);
}
```

上面的两种写法是等价的。不难看出，$g(x)$在这里起到的作用就是改变在a, b之间的平滑的插值的量。

在ShaderToy中最常看到的，一个典型的写法可能是这样的（二次kernel函数）:

```glsl
float smin(float a, float b, float k) {
    // k *= 4.0; 
    // 这一行在很多ShaderToy中没有添加，这里对应着`k /= g(0)`的一行，是对平滑因子的归一化。
    // 所以很多ShaderToy的smin函数的k参数其实是不太合理的，下文会解释
    float h = clamp(0.5 + 0.5*(a-b)/k, 0.0, 1.0);
    return mix(a, b, h) - k*h*(1.0-h);
}

// or, more effeciently, as below:

float smin( float a, float b, float k )
{
    k *= 4.0;
    float h = max( k-abs(a-b), 0.0 )/k;
    return min(a,b) - h*h*k*(1.0/4.0);
}
```

上述两种写法是等价的，对应$g(x) = (\frac{x+1}{2})^2$的情况，可以拆开自行推导一下。之所以后者的写法更高效（也更推荐），是考虑到GPU结构的优化所致，因为很多单一函数本身并不能保证$g(x)$需要的性质，所以$g(x)$往往是在$(-1, 1)$区间内是一种形式，而$[1, +\infty)$和$(-\infty, -1]$则要用其他形式（具体来说，就是$g(x) = x$和$g(x) = 0$）的分段函数。这就不可避免的要使用`clamp`或者分支结构。而GPU程序在执行这种分支结构的逻辑时效率会更低，要充分利用GPU性能最好将逻辑中的分支“flatten”成不管输入为何，可以统一串行执行下去的代码[^1]。

<video loop autoplay controls>
  <source src="https://iquilezles.org/articles/smin/vid_00.webm" type="video/webm">
  视频可能挂了..
</video>

视频中的蓝色边框代表着从各自的shape SDF中向外延伸k个单位（归一化后的）的”inflate“ shape（也叫成bounding volume expansion），可以说只要他们的inflate shape发生接触，就会产生融入的效果。
为什么要先把传入的参数k进行一个变换呢？为什么又偏偏是要除以$g(0)$呢？

<iframe width="640" height="360" frameborder="0" src="https://www.shadertoy.com/embed/XfdSWj?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe>

首先搞明白一点，k是我们在对shape归一化之后希望inflate shape扩展的距离。如果没有对k的归一化操作，当$x = 0$也就是$b = a$时，smin函数的返回值为$b - k * g(0)$。而此时我们希望得到的应该是
$b - k$，因为a都和b相等了！如果这个点恰好是infalte shape扩展后相切的点，那么它到某个shape的真实距离d应该恰是$a + k$（也就是$b + k$）。因此我们应该希望这种情况下smin恰好返回$b - k$，这样当实际距离为$b + k$时，我们从smin处得到的平滑后的距离就是b，这样对构建SDF的过程来说就是透明的了！

为此，我们需要将k先除以$g(0)$，这相当于对平滑因子做了归一化。在上面的shadertoy示例中，可以用鼠标点击查看某个鼠标点计算出的平滑后的距离的圆，而红色/绿色圆对应着两个shape的inflate shape。

## smin的常用kernel函数及应用

(文中的ShaderToy组件显示上有些问题，播放按钮没能完全显示，鼠标移动到组件上，在重置按钮下面点击播放按钮即可)

常用的smin函数可以参考CD Family[^2]的部分。在iq的博客中，将一些拥有更好特质的kernel函数叫做Clamped Differences Family函数，正如其名，不只要求x趋近于无穷时，kernel的值趋近于x或0，而是要求从-1 和 1开始就要明确的分界，也就是$g(x) =\left\{ \begin{array}{l}x, x\in[1, +\infty), \\ 0, x\in(-\infty, -1) \end{array} \right.$；不仅如此，还要做到在-1和1处连续，也就是$g'(1) = g(1) = 1, g'(-1)  = g(-1) = 0$。这样的kernel函数可以保证在$abs(b - a) > k$时，smin函数不会有任何影响。

下面的shadertoy对应着各种kernel函数的表现。

<iframe width="640" height="360" frameborder="0" src="https://www.shadertoy.com/embed/DlVcW1?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe>

（如果不会动请参考<https://www.shadertoy.com/view/DlVcW1>)

最直观的smin应用莫过于实现SDF图形之间的“融合”(melt)效果。

<iframe width="640" height="360" frameborder="0" src="https://www.shadertoy.com/embed/4scXRB?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe>

此shader便是使用了二阶的平滑kernel函数。


[^1]:<http://www.aclockworkberry.com/shader-derivative-functions/>
[^2]:<https://iquilezles.org/articles/smin>
