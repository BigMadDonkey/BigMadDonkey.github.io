---
layout: post
title: Unity Shader学习入门
categories: [Blog, Unity, Shader]
description: 本篇是对Unity Shader的学习博客。
keywords: Unity, shader
---

{:toc}

## 引言

本篇是学习Unity下图形渲染Shader编写的博客。

首先我们必须明确一点：Unity下支持3类Shader的编写：图形渲染Shader，会在GPU的渲染管线中得到执行；Compute Shader，用来利用GPU进行复杂计算；Ray Tracing Shader，用来处理光线追踪。

而本文对Shader的学习主要是基于图形渲染Shader，以下简称Shader。在Unity中，Shader的使用形式为：为某个Material所持有，该Material不仅可以持有Shader，还可以持有自己特定的Shader相关属性。不同Material可以共用一个Shader，但是设置不同的属性。

👀不过更准确来说，根据Unity官方文档的说法，实际上我们在Unity中操作的Shader应被称作“Shader Object”，它内部拥有多个Shader Program，可以根据情况不同选择不同的处理方式，还持有Shader Peoperty，可供外部传入。我们在Unity中创建的.shader 文件或是Shader Graph，严格来说都属于Shader Object，它们实际上是对Shader Program的一个封装。

此外，还需明确的一点是关于编写Shader使用的语言。网上有些比较老旧的教程常常把CG与HLSL混用，但是CG很早就不再受官方支持了，Unity官方指定了编写Shader的语言：
1. ShaderLab: 用来组织起基本的Shader Object的结构，例如一个.shader文件的外层结构语法基本都是ShaderLab. 更详细的介绍可以参考下文的部分。
2. HLSL: 用来编写Shader Object中的Shader Program，也就是各个SubShader的部分。

🤔读者可能觉得奇怪：HLSL不是为DirectX 所专有的Shader语言吗，为什么现在都指定使用HLSL来编写Shader Program？其实这个问题很简单：对于目标DirectX的平台，就使用HLSL相应的compiler来编译Shader即可；对于目标图形API为OpenGL的平台，则在此基础上，将HLSL编译后的字节码通过[HLSLcc](https://github.com/Unity-Technologies/HLSLcc)翻译成GLSL的字节码。其他目标平台Metal、Vulkan等也是类似的。
> 当然你要是想直接用GLSL什么的也不是不行，但是为了遵循标准workflow，建议还是使用HLSL来编写Shader Program。




## Shader的结构

一个Shader文件，即Shader Object的通常结构如下图所示：
![Shader Structure](/images/blog/UnityShaderLearning/UnityShaderLearning-shader_structure.png)

图中看到的部分都属于ShaderLab的范畴。其中最外层由一个`Shader`关键字加上命名构成，内部由一个整体的花括号涵盖。在旧版本（5.0之前）中，Shader的命名会影响Shader的效果，可参考[Legacy Shader](https://docs.unity3d.com/2021.3/Documentation/Manual/Built-inShaderGuide.html)。新版本Unity下创建的非legacy Shader的命名则没有特殊作用。

在`Shader`块内部，分有4大类成员：
1. Properties: 存储那些由Material设置的属性参数，可以通过Unity inspector来设置Material选择的Shader已经对应的Shader参数。
 
   Property的标准声明格式为：
   
   `[optional: attribute] name("display text in Inspector", type name) = default value`
   
   在开头可以添加可选的attribute来表明这一Property的特性。attribute支持的选项可见于[Material-Properties-attibutes](https://docs.unity3d.com/2021.3/Documentation/Manual/SL-Properties.html#:~:text=and%20shaders.-,Material%20property%20attributes,-Material%20property%20declarations)。
   Properties块是一个可选项。

2. CustomEditor: 这一项是用来自定义Inspector中数据的编辑方式的。有时默认的Inpsector不够用，那么可以引入自己实现的继承自`ShaderGUI`的C#类，并将脚本放置于Assets/Editor下，然后在Shader中可以通过CustomEditor 传入自己的类名，来指定使用自己编写的GUI来操作数据。值得一提的是ShaderLab还支持`CustomEditorForRenderPipeline`块，允许根据不同的SRP来选用不同的CustomEditor。如果同时存在这两种关键字，在满足特定的SRP的条件下，后者会覆盖前者。CustomEditor块是一个可选项。**CustomEditor**必须定义在Properties之后，否则编译无法通过。

3. SubShader: 该块是定义Shader Program的位置。一个Shader Object中可以有多个SubShader，且至少要有一个。定义多个SubShader时，会视情况选择合适的SubShader。

4. Fallback: 如果Unity判定当前Shader Object无法使用，则可以使用Fallback指定的Shader。Fallback块是可选的，也可以设置为`off`。

> 吐槽下Unity官网的Shader教程，关于CustomEditor一节的示例是完全错误的。首先CustomEditor声明时不需要等号，其次，CustomEditor必须定义在Properties块之后，否则编译不通过，这跟官网给出的示例完全不一致：
> ![官网上的错误示例](/images/blog/UnityShaderLearning/UnityShaderLearning-offical_site_wrong_example.png)
> 真不是我说，官网上的文档示例最起码也要验证一下吧，你这连编译都不通过，弄的我对其他文档信息都开始怀疑了，最后还是Chat GPT告诉我定义的顺序有问题...(大怨种.gif)

## SubShader

SubShader的内容是GPU实际要执行的程序。由于可以定义多个SubShader的性质，使得可以根据不同硬件/Render Pipeline/图形API来设置不同的GPU setting，或是执行完全不同的Shader Program。

我们还是先来看下SubShader的基本结构：


### Tags

Tags是一系列键值对，对于一些与定义的key，可以设置对应的值来影响Unity的渲染工作，参见[Unity Shader Tags](https://docs.unity3d.com/2021.3/Documentation/Manual/SL-SubShaderTags.html)。也可以自定义自己的Tags，在C#侧，可以通过`Material.GetTag`实例方法来获取指定key的Tag值。

SubShader和Pass中都可以定义Tags，但他们并不互通，在Pass中使用SubShader的外层Tags并没有效果，反之亦然。

### LOD

SubShader中的LOD与渲染网格中的LOD概念类似，都是代表着细节层级（Level of details），会根据需要LOD的不同动态切换。不过，SubShader的LOD并不会自动计算，需要手动设置，而且跟物体到相机的距离没有任何关系。

由于Unity在选取SubShader时，是按照SubShader在文件中定义的顺序去检测的，因此需要将SubShader按照LOD降序排列，才能确保选择满足低于maxLOD的最好Shader，否则可能选到了一个较低等级的SubShader。

### Pass

Pass中可以存放设置GPU状态的指令，和实际要执行的Shader Code。Pass可以指定名字和自己的Tags。


## HLSL语法学习

