---
toc: true
title: 深入理解weakStrongDance
date: 2020-04-11 18:29:57
categories:
tags:
---

## 前言

`weak strong dance`几乎是每个iOS开发者必备的技能，几乎大多开发者都知道在block中进行strong是为了引用计数加1。 保证对象不会再Block执行中被释放。但是问题真的这么简单吗？



## weak指针

关于weak的实现就不详述了，谈一谈weak指针是如何被使用的。 我们知道OC调用方法实际是对对象发送msgSend。但是对于weak指针却不是这么单纯。 

