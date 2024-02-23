---
layout: post
title: 为VSCode配置自定义的语法token配色
categories: [VSCode]
description: 为VSCode配置自定义的语法token配色
keywords: VSCode
---

{:toc}

## 引言

久违地更新一下，这次是自定义修改VSCode语法高亮的颜色的方式。乍一看这是个很简单的需求，但其实也有些技巧在里面。

## 配置颜色的方式

在setting.json中，可以创建`workbench.colorCustomizations`属性，来覆盖当前主题的配色。里面可以设置编辑器里几乎一切颜色，例如按钮、侧边栏背景等。无论如何，这不是今天的重点。

要修改语法token的配色方式，则要通过创建`editor.tokenColorCustomizations`的属性。

## tokenColorCustomizations 写法

一个典型的写法可能是这样的:

```json
"editor.tokenColorCustomizations": {
    "[Monokai]": {
        "comments": "#229977"
    },
    "[*Dark*]": {
        "variables": "#229977",
        "textMateRules": [
            {
                "scope": "meta.function.lua keyword.control.lua",
                "settings": {
                "foreground": "#ef5bb9"
                },
            },
        ],
    },
    "[Abyss][Red]": {
        "keywords": "#f00"
    },
    
}
```

首先，支持按照Theme Color的id匹配来决定对不同的主题如何应用颜色的override。例如`[*Dark*]`就匹配了那些名字里带Dark的主题。
在其内部，可以声明几类简单的颜色覆盖方案：

```json
{
    "keywords": "#c663f8",
    "comments": "#577b89",
    "functions": "#FF0000",
    "variables": "#FF0000",
    "numbers": "#FF0000",
    "strings": "#FF0000",
    "types": "#FF0000",
}
```

这些是比较简单的覆盖。

然而，有时可能需要给特定的semantic token设置value，能不能做到呢？当然可以！读者可能注意到先前的示例中有出现`textMateRules`属性，其正是对特定semantic token进行样式修改的方式！在`textMateRules`中，可以指定多个修改，每个修改需要指定一个scope表示应用范围。在settings中可以指定要修改的具体属性。

但是怎么知道我要修改的那些东西被语法解析成什么样的token呢？换言之，它们的scope？不用担心，因为VSCode有很方便的工具：聚焦在某个编辑器中，然后执行如下的命令：

![执行此命令](/images/blog/VSCodeTokenColor/VSCodeTokenColor-execute_command.png)

好了！现在，只需要将光标放在你想要设置颜色的token中，你应该能看到这个token的相关scope属性！

![token scope](/images/blog/VSCodeTokenColor/VSCodeTokenColor-token_scope.png)

要注意的是，一个token可能会有多个（大多数情况下也确实会有多个）scope，并且scope有优先级之分，越详细的scope其优先级越高。然而，也可以在定义`textMateRules`时，指定多个scope，那么这个规则会作用于同时满足所有scope的token（逻辑与）。

例如，要覆盖上图对`function`关键字的样式，你应该定义如下的rule:

```json
{
        "scope": "meta.function.lua keyword.control.lua",
        "settings": {
        "foreground": "#ef5bb9"
        }
    },
```

就这样。后面有新发现，也许会更新。
