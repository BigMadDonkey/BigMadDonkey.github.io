---
layout: post
title: Lua中metatable相关的小记
categories: [Blog, Lua]
description: Lua的metatable相关的注意点。以后也许还会更新也说不定。
keywords: Lua, metatable
mathjax: true
---

{:toc}

## 引言

本篇是对Lua 元表机制的学习记录。以后如果再发现新的注意点可能还会更新。主要参考了[Lua 5.4 Reference Manual](http://www.lua.org/manual/5.4/manual.html)，并在其提供的在线Demo界面在线运行Lua代码来验证。


## metatable简介

Lua作为一个脚本语言，与JavaScript存在许多共同点，无论是弱类型、解释执行，还是语法层面，它们都很相似。在OOP方面的设计思想也十分类似：它们都不直接提供对类的支持，大部分事物都是对象，在JavaScript中，想要达到继承的效果需要依赖prototype链，在Lua中则是依赖于元表。不过，JavaScript也提供了语法层面的对类的支持（虽然其实还是prototype的语法糖），这可比只能支持基于对象的继承的Lua要好多了。如果使用Lua，就不得不自己实现一套类的继承机制...
<br />
<br />
<br />
跑题了。关于metatable和prototype的另一个不同点在于对访问父层次成员能力的要求。在JS中，只要对象之间建立起prototype的关系，不需要任何其他操作，当尝试在对象上访问其不存在的成员时，会沿着原型链自动往上查找；而在Lua中，则要求metatable必须带有`__index`或`__newindex`成员。它们分别就决定了读取/写入字段时，如果table上不存在该字段，将如何从metatable上来访问。

在Lua中，通过`setmetatable`和`getmetatable`来访问table的元表。

## __index与__newindex

Manual中这样描述`__index`:
> __index: The indexing access operation table[key]. This event happens when table is not a table or when key is not present in table. The metavalue is looked up in the metatable of table.
> 
> The metavalue for this event can be either a function, a table, or any value with an __index metavalue. If it is a function, it is called with table and key as arguments, and the result of the call (adjusted to one value) is the result of the operation. Otherwise, the final result is the result of indexing this metavalue with key. This indexing is regular, not raw, and therefore can trigger another __index metavalue.

两个要点：
1. 只有当被查找的元素不是table，或是table中不存在要查找的key时，才会使用`__index`来根据元表向上查找。
2. `__index`的可取值可以是函数，或是另一个table，或是其他带有`__index`的值。所谓的metavalue是指`__index`这样的，元表特有的一系列特定成员。如果`__index`是函数，则会将table和key作为输入，执行函数，返回值作为`__index`的访问结果（相当于getter?）；否则相当于在`__index`上访问key成员。


关于第一条，难道不是table也可以有metatable吗？答案是肯定的：你可以随便创建一个string，在其上调用`getmetatable`就知道了。不过，虽然其他类型有的也会有自己的metatable，但如果是想要自行通过`setmetatable`来给非table类型添加metatable，则会报错。只有使用debug library才可以为非table类型修改他们的metatable。

关于第二条，这决定了通过元表也可以继承式地向上不断访问。`__index`的值完全可以是一个表，一个getter，或是另一个带有`__index`的表。

而对于`__newindex`也是类似的，两个要点对它也适用，区别在于`__newindex`作用于setter的场合。



如果不管metatable，就是想明确地访问table本身的内容，可以使用`rawget`和`rawset`函数。

## __metatable

根据官方Manual，调用`getmetatable`时，如果metatable有此字段，则会返回该值；否则返回metatable本身。

> If object does not have a metatable, returns nil. Otherwise, if the object's metatable has a __metatable field, returns the associated value. Otherwise, returns the metatable of the given object.

`__metatable`字段究竟起到了什么作用？难道获取metatable时，不是直接把元表返回了就行吗？

我们不妨看看`setmetatable`的doc：

> setmetatable (table, metatable)
> 
> Sets the metatable for the given table. If metatable is nil, removes the metatable of the given table. If the original metatable has a __metatable field, raises an error.

通过`setmetatable`修改一个table的metatable时，如果其原来的metatable有`__metatable`字段，则会报错。`__metatable`不仅决定了通过`getmetatable`可以获取的返回值，还使得不能通过`setmetatable`更改元表！

```Lua
local super = {__metatable = true}
local derive = setmetatable({}, super)
-- 以下代码会报错：
-- cannot change a protected metatable
setmetatable(derive, {})
```

所以`__metatable`的设置还起到了禁止更改metatable的作用。这保护了table的状态，确保table的元表不会被外部更改。

```C
\\lbaselib.c -- from lua/lua github repository
static int luaB_setmetatable (lua_State *L) {
  int t = lua_type(L, 2);
  luaL_checktype(L, 1, LUA_TTABLE);
  luaL_argexpected(L, t == LUA_TNIL || t == LUA_TTABLE, 2, "nil or table");
  if (l_unlikely(luaL_getmetafield(L, 1, "__metatable") != LUA_TNIL))
    return luaL_error(L, "cannot change a protected metatable");
  lua_settop(L, 2);
  lua_setmetatable(L, 1);
  return 1;
}
```
可以在Github上的Lua仓库中找到setmetatable相关的代码。`LUA_TNIL`是常数0，代表着Lua虚拟机中的nil。<aside>强力推荐 sourcegraph chrome扩展，在线看github开源仓库巨方便！最近还继承了cody AI助手，可以直接让他看仓库中的某些文件，响应很快，比chat gpt专业一些。虽然也有胡编的毛病，但总体还是很方便的！</aside>