---
layout: post
title: Phong光照模型
categories: [CG]
description: 本篇是对Unity Shader的学习博客。
keywords: Phong Reflection Model, Lambertian Reflection
mathjax: true
---

{:toc}

## 引言

本篇博客介绍Phong反射模型，并总结个人观点。主要参考维基百科页面。

Phong的光照反射模型与1975年由裴祥风提出，是基于对现实光照的观察经验得到的模型，并已成为现代计算机图形学的光照模型基础，

需要注意的是，下文介绍模型中常常会提到“某某夹角的余弦值”，用来计算光照，但实际上只有在余弦值 $\geq$ 0时才有意义，毕竟光不可能透过表面给你来个负方向的分量抵消了吧！


## Phong模型

Phong模型将光照在物体表面的反射分解成三部分：
1. 环境光(Ambient): 就算没有光直接照到表面，也会有一个环境光，代表着散布在场景中的全局的少量光。环境光通常反映为一个常量（由一个环境光反射系数控制）。
2. 漫反射(diffuse): 光照到物体表面时沿各方向发出各向同性的光，它们的强度相同。实际上参考了Lambertian的漫反射模型（见下文）。
3. 镜面反射(specular): 光照到物体表面时，与表面法向量对称地反射出出射光。

Phong发现，现实中表面粗糙的物体，其表面的反光明亮区域总是大而偏暗，而表面光滑（闪亮）的物体其反光明亮区域（亦即highlight）则是小而明亮。基于此它引入了一个额外的参数来表示表面的粗糙程度，我们在后文的数学表达部分介绍。

实际使用中，往往将RGB三个通道分别进过Phong模型处理。因此，实际使用中可以做到对不同的通道使用不同的模型参数。

### 漫反射的兰伯特模型

Phong模型对漫反射的处理实际上沿用了Lambertian Reflection的方法：其将一点的光线在物体表面上一点向各个方向的漫反射光视为各向相同的，并且漫反射的强度仅与入射光强度、入射光线与顶点法向量的夹角余弦值和漫反射系数有关。如果入射光强度一定，入射光与定点法向量夹角越小，漫反射光的强度越高，当入射光与顶点法向量垂直，则完全无漫反射光。

![LambertianReflection](/images/blog/PhongLightingReflectionModel/PhongLightingReflectionModel-LambertianReflection.png)

如图所示，其中 $\vec{N}$是顶点法向量，$\vec{L}$是从顶点指向光源的向量。

这一模型大大简化了漫反射的模拟工作。实际计算中这两个向量往往都先经过归一化，因此它们的点积就是$cos\alpha$。

### Phong模型的数学表达

通过前面的介绍我们知道，环境光和漫反射的计算都相对简单：不论观察点位于何处，环境光都是一定的，而Lambertian漫反射由于各向同性，也是一定的。关键在于如何计算镜面反射的强度。

参考下图：

![Blinn-Phong Model](/images/blog/PhongLightingReflectionModel/PhongLightingReflectionModel-Blinn_Phong_Model.svg)

其中，$L$是由入射点射向光源的向量，$N$是入射点的表面法向量，$R$是镜面反射光的向量，$V$是入射点到观察点的向量。下文中这些向量上带小hat，表示它们经过了归一化。$H$向量这里还用不到，我们暂时忽略即可。

对于环境光，计算起来很容易：我们首先有环境光强度$i_a$和环境光反射系数$k_a$。它们的乘积$k_a i_a$就是环境光部分的分量。不论光源怎么变动，入射角、观察点怎么变，都是一样的。

对于漫反射，根据前文Lambertian Reflection一节，可以知道对于一个光源m，他在某一点的入射光的漫反射强度可以表示为$k_d(\hat L \cdot \hat N)i_d$，其中$k_d$表示漫反射的反射系数，$\hat L \cdot \hat N$实际上就是入射光与法向量夹角的余弦值，$i_d$则是光源的漫反射分量的强度。

对于镜面反射，我们需要根据图片判断：首先，得到镜面反射向量$R$的强度，然后在视角方向$V$上做投影，即是从某一点观察镜面反射得到的分量强度。镜面反射部分分量可以表示为$k_s(\hat R \cdot \hat V)^\alpha i_s$，其中$k_s$是镜面反射的反射系数，$\hat R \cdot \hat V$表示镜面反射光与视角方向的余弦值，$i_s$表示光源的镜面反射分量强度。实际上就是起到了将镜面反射向量投影到视角方向上。不过，其中还有一个$\alpha$指数在夹角余弦值上，这里起到了什么作用？

实际上，这就涉及到Phong模型对粗糙表面和光滑表面高光区域的理解：粗糙的表面，他的高光区域要大而偏暗，光滑的表面的高光区域要小而发亮。为此，需要确保引入一种参数，能够实现当表面光滑时，高光区域在镜面反射向量方向附近可以观察到，而偏离一些就几乎看不到，**换言之让镜面反射的"可视区域"(光强足够大，人眼可见)偏小**，这也就要求对于视角$V$和反射向量$R$夹角偏大的情况，要快速地让看到的镜面反射光强分量削减。而$\alpha$正是为此而引入的。$\alpha$是一个与物体表面材质相关的参数，代表着物体的光滑程度。由于夹角余弦值一定在$(0, 1)$之间(小于0则根本不考虑),对于相同的夹角，$alpha$越大，$(\hat R \cdot \hat V)^\alpha$的分量就越小，最终导致镜面反射部分的分量越小，也就是能看到镜面反射光的视角可以偏离反射光线方向的范围越小，最后呈现的效果就是高光区域越小。因此，**$alpha$可以认为是材质的光滑系数，$\alpha$越大，代表物体表面的高光区域越小！**

> 高光区域小，但是反射光可能更亮——这就取决于如何设定$k_s$了。

最后要记得：漫反射和镜面反射对每一个光源都要计算。我们可以得到最终的Phong模型光照反射的数学表达形式：

$$
I_p = k_a i_a + \sum_{m \in lights}(k_d \hat L_m \cdot \hat N) \cdot i_{m,d} + k_s(\hat R_m \cdot \hat V)^\alpha i_{m,s}) 
$$

其中m是场景中的光源。

## Blinn-Phong模型

1977年Jim Blinn对Phong模型提出了一些改良。这主要是因为计算镜面反射的向量$R$比较麻烦（需要沿着法线对称），他提出一种做法：使用$L$与$V$的向量和与$N$的夹角来替代$V$与$R$的夹角。

让我们再看一次Phong模型的图解：

![Blinn-Phong-Model](/images/blog/PhongLightingReflectionModel/PhongLightingReflectionModel-Blinn_Phong_Model.svg)

其中$H$就是$L + V$得到的方向上的向量，由于实际上就是$L$和$V$夹角角平分线上的向量，也称作半程向量(Halfway)。$N$与$H$的夹角就是$R$与$V$夹角的一半。

实际使用中，我们用 $\frac{L + V}{\|\|L + V\|\|}$ 来计算$\hat H$，用$\hat H \cdot \hat N$代替原本的$\hat R \cdot \hat V$。不过单纯替换这一部分，结果会与原来不同，因为夹角只是原来的一半，要达到跟Phong模型类似的效果，必须从其他方面确保高光区域大小基本不变。对于相同的夹角，通过半程向量得到的余弦值现在会比原来偏大，因此需要设法让结果偏小。Blinn模型的做法是调整材质的系数，例如令$\alpha{'} = 4\alpha$，来大致接近Phong模型的效果。

> 根据维基百科的说法，虽然Blinn-Phong模型是为了减少计算量的同时接近Phong模型的结果，但实际上Blinn-Phong模型在很多情况下比Phong模型得到的结果更符合经验。



