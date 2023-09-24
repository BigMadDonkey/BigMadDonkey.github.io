---
layout: post
title: 为VSCode配置Godot调试环境
categories: [Godot]
description: 介绍为Godot配置Debug环境的笔记。
keywords: Godot
---

{:toc}

## 引言

本篇介绍在使用VSCode的场景下，如何配置用于Godot Debug的C#和GDScript的调试环境。注意，博客发布日期为2023.09.24，所用Godot版本为4.1，目前VSCode的相关插件适配的都不是很好，读者阅读这篇博客时可能这两个插件已经对Godot4.x版本有了较好的适配。**请读者注意博客的时效性**。

无论使用哪种方式，在Godot Editor，都需要先做一些设置：

* 打开调试->使用远程调试部署
* 编辑器->编辑器设置->网络->调试适配器->同步断点

## 配置GDScript Debug环境

安装`godot-tools`插件，并对扩展做如下配置：

* Godot_tools: Gdscript_lsp_server_port: 设置为Godot的LSP端口，可见于Godot Editor 编辑器->编辑器设置->网络->语言服务器->远程端口。
* Godot_tools: Editor_path: 设置为Godot Editor可执行程序的路径，这个看你自己了。把Godot的可执行程序路径加入环境变量中，可以简化这里的内容。

然后，按照扩展介绍的提示，生成一份launch文件，并修改生成的"GDScript Godot"配置下的如下字段：

* "port": Godot DAP的端口，可见于Godot Editor->编辑器->编辑器设置->网络->调试适配器->远程端口。
* "debugServer": 内容同上（注意，这个字段并没有自动生成，但是这个字段在Godot4.x下是需要的）。
* "address": "tcp://127.0.0.1"，主要是要在前面加上协议。

好了，现在可以试试了，在VSCode侧给代码设置断点，然后F5运行！

### 体验小结

该插件对Godot 4.x的支持不够。即便好不容易成功配置好了，GDScript的Debug也十分受限，很多运行时内容看不到，监视窗口也不能执行表达式，可以说是很难用了！

## 配置C# Debug环境

安装`C# Tools for Godot`插件，并按照扩展提示生成launch和tasks文件。

别看launch生成了许多config，描述的头头是道...实际上经本人测试，Godot4.x下这里的配置都不好用，会莫名奇妙地报错，或者是自己断开。

经过反复试错和查询issue，最终找到一种勉强可行的调试方式。参考[这位开发者的答复](https://github.com/godotengine/godot-csharp-vscode/issues/43#issuecomment-1258321229),只需要修改一部分配置即可：

launch.json的部分，基本可以复制原答复，只是program需要根据读者的实际情况调整。例如，我自己的launch.json新增内容如下：

```json
{
   "name": "Play",
   "type": "coreclr",
   "request": "launch",
   "preLaunchTask": "build_dot",
   "program": "godot",
   "args": [],
   "cwd": "${workspaceFolder}",
   "stopAtEntry": false
  },
```

因为我以配置了godot的环境变量，并修改了可执行程序的命名，所以program可以直接用"godot"。preLaunchTask的内容则要跟下面新增的Task命名相匹配：

```json
{
    "label": "build_dot",
    "command": "dotnet",
    "type": "process",
    "args": [
      "build"
    ],
    "problemMatcher": "$msCompile",
    "presentation": {
      "echo": true,
      "reveal": "silent",
      "focus": false,
      "panel": "shared",
      "showReuseMessage": true,
      "clear": false
    }
  }
```

然后就可以愉快地F5了，好耶！

### 体验小结

C#的这个扩展基本上完全没什么用，除了创建配置文件方便一点。实际上只是完全用dotnet 执行构建然后godot打开而已。不过C#的debugger监视窗口可以看的东西比较全面，这算是一个优点。
