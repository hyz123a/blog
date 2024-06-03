---
title: C++ 的多态 
categories: Dev
tags: [Linux, C++, 多态]
created: 2024-5-29 23:18:00
---

最近经常能够看到 C++ 面经中出现 C++ 类的内存布局及初始化一个派生类时基类，派生类的构造顺序等这样的问题。同时自己也在这方面存在疑惑，所以就专门去学习了相关的知识，并在查找资历的过程中记录一下。

## C++ 类内存布局

在开始之前先看一下 clang 的两个命令：
```bash
c++ -Xclang -fdump-record-layouts xxx.cpp
c++ -Xclang -fdump-vtable-layouts xxx.cpp
```

## C++ 类的构造顺序

**带参数的构造函数，在存在默认初始化的成员变量的情况下**

