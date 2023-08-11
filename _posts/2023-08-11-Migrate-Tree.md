---
title: "Migrate Tree to Lolly"
categories:
  - blog
tags:
  - lolly
  - mogan
---

#  核心结构Tree和tree_label分离并迁移Tree
-----------

链接：

- [remove tree and use lolly v1.1.0](https://github.com/XmacsLabs/mogan/pull/929) 
- [fix separate modification and tree_label](https://github.com/XmacsLabs/mogan/pull/927)
- [Revert ispell changes on tree label](https://github.com/XmacsLabs/mogan/pull/925)
- [separate observer and observers](https://github.com/XmacsLabs/mogan/pull/921)
- [separate observer and tree_label](https://github.com/XmacsLabs/mogan/pull/920)
- [separate modification and tree_label](https://github.com/XmacsLabs/mogan/pull/919)
- [seperate tree and tree_label](https://github.com/XmacsLabs/mogan/pull/918)

## 背景

接上文，由于需要精简mogan中的代码，方便用Wasm做浏览器版本，所以分离了mogan的基础设施，组成Lolly仓库，也就是mogan的`L1 kernel`。而`tree_label`中定义的枚举类型和文档有很强的相关性，所以需要将`tree`和`tree_label`分离。本文将继续介绍理清文件依赖并拆分代码的过程。

## 计划

考虑把`Tree`操作相关的代码放到`tree_helper`中，将`tree_label`中的`enum`类型继承自`int`，使其在不同编译器上推导的内存占用一致，统一为4byte。

然后将`tree`中需要传递`tree_label`类型的所有函数接口统一改成int类型传参，让编译器编译时执行隐式转换。

## 开始动手

**首先拆`tree`和`tree_label`：**

迁移至`tree_helper.hpp`中的代码如下：

```C++
// tree_helper.hpp
inline tree_label L (tree t) {
  return static_cast<tree_label> (t->op);
}
inline tree_label& LR (tree t) {
  return *(tree_label*)(&(t->op));
}
inline string get_label (tree t) {
  return is_atomic (t)? t->label: copy (as_string (L(t)));
}
```

这边有一处细节，这个LR函数原本是这样写的：

```C++
inline tree_label& LR (tree t) {
  return t.rep->op; }
```

它返回的是对这个`tree_label`的引用，当我们把该`op`改为`int`类型时，给它返回一个`tree_label`类型的引用有一些困难。

==问了tangdouer，给出了如上的一种写法，这里解释一下：==

> 首先获取了tree中op的地址，然后将其强制转换为`tree_label`类型的指针，最后解引用该指针，返回`tree_label &`类型的值

**拆分`modification`和`tree_label`：**

主要坑点是这边：

```C++
// tree_helper.hpp
tree_label L (modification mod); 

// tree_helper.cpp
tree_label
L (modification mod) {
  ASSERT (mod->k == MOD_ASSIGN_NODE, "assign_node modification expected");
  return L (mod->t);
}
```

这里重载了L这个函数，对原本的`get_label`做了封装

**拆`observer`和`tree_label`：**

主要是将`observer`中和tree有关的函数提取到`tree_observer`中，然后分离`observer`

## 总结

虽然是迁移代码，但也算自己的第一次代码重构。

