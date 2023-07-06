---
layout: post
title: 不同的颜色格式
categories: Blog
description: 使用调色板等工具输入颜色时，你有仔细了解过它们底层使用的颜色格式吗？
keywords: frontend, CSS, color-format
mathjax: true
---

{:toc}

## 引言

有计算机常识的人都应该知道常见的显示器显色原理都是通过Red、Green和Blue三基色合成一个单一像素点。在某些需要设置简单颜色的场合，最常使用也最为人所熟知的应当是RGB格式了——将RGB三种颜色划分成0～255个具体的显示强度，三个通道的颜色合成在一起就是一个色点。再特别一点，就是还支持透明度通道。

可是，不知道读者是否好奇过，在各种场合使用调色板时，颜色的format就没有RGBA以外的形式了吗？

![调色板](/images/blog/DifferentColorFormat/DIfferentColorFormat-palette.png)

这种在一个平面内取点的情况，怎么想也不是用3个字节能表示的吧！

## HSL、HSB(HSV)和HSX

实际上这种**选定一个色调，然后调整其颜色程度**的格式，叫做HSL（Hue色调、Saturation饱和度、Lightness亮度）。
这其中：

* Hue: 色调，决定了选择的颜色基调，取值范围为0～360（360实际上与0一致），就像一个圆，在一个圆的范围内取色，实际上其单位也是degree(在CSS中，缩写为deg)。
* Saturation: 饱和度，决定了该色调的鲜艳程度，可取值范围从0～100%。为0时不带任何颜色，为100%时表示最饱和状态。
* Lightness: 亮度，决定了颜色的亮暗程度，可取值范围0～100%。取0时，表示完全黑暗，一片黑；取100%时，表示完全光明，一片纯白。

![HSL调色板](/images/blog/DifferentColorFormat/DIfferentColorFormat-HSL.png)

从图中不难看出当L取最高时，色调、饱和度无论如何调整，都是一片纯白；L取最低时则是一片纯黑。因此，HSL格式下若想输出某种“纯色”格式，正确的做法是选择好色调，并将S拉满、L则置于50%。HSL格式的特点是，调色板的纯色区域附着在L的中间位置。

此外，还有一种与HSL相近的格式：HSB（亦称HSV），它们唯一的区别在于明亮程度的表示方式。

HSB格式中使用Brightness来代替Lightness。虽然它们都起到表示色彩明亮程度的效果，但Brightness在取值为100%时，代表着该颜色下最亮的纯色，而要达到类似的效果Lightness需要取值为50%。

![HSL vs HSB](/images/blog/DifferentColorFormat/DIfferentColorFormat-HSLvsHSB.png)

观察上图不难发现，纯色区域在HSB模式下偏向Brightness = 1的区域，而在HSL模式下偏向Lightness = 0.5的区域。

从给定的HSL格式颜色，可以转换到RGB的格式，公式如下：

$$
H' = \frac{H}{60^\circ} 
X  = C \times (1 - |H' mod 2 - 1|)
$$


## YUV

YUV色彩格式主要应用于视频、图像的编码传输（例如JPEG/MPEG）。由于人眼👀对亮度更敏感，而对色彩信息相对不敏感，因此在媒体的流式数据传输时，考虑到带宽占用和处理能力，可以牺牲一些色彩上的信息而优先保存亮度信息，减少编码量的同时不会对视觉效果有太大影响。

其中，Y表示亮度，U和V分别表示两种色度。最详细的格式，就是为每个像素点保存一个Y分量，一个UV分量，但是正如上文所说，Y分量往往比较重要，而UV分量存储地如此详细则显得没有必要。

YUV格式有诸多变体，但它们都可归类于YUV格式（亮度与色度分离存储）。

### 采样格式

YUV格式存在多种采样方式，决定了如何对UV分量进行采样。在考虑采样格式时，请注意：无论如何，每个像素都要有Y分量。Y分量是不能缩减的。
   我推荐一种方法：可以设想一个4x2的矩阵，共有8个1x1的像素点需要保存信息，不同的采样格式决定了


1. YUV 4:4:4 格式：每个Y分量对应一个完整的UV分量，这说明色彩没有损失。
2. YUV 4:2:2 格式：