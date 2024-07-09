---
title: C++ 的类
categories: Dev
tags: [Linux, C++, Class] 
created: 2024-5-29 23:18:00
---

最近经常能够看到 C++ 面经中出现 C++ 类的内存布局及初始化一个派生类时，基类，派生类的构造顺序等这样的问题。同时自己也在这方面存在疑惑，所以就专门去学习了相关的知识，并在查找资历的过程中记录一下。

## C++ 类内存布局

在开始之前先看一下 clang 的两个命令：
```bash
clang++ -Xclang -fdump-record-layouts xxx.cpp
clang++ -Xclang -fdump-vtable-layouts xxx.cpp
```

对于代码

```C++
#include <iostream>

class A {
public:
  A(int x) {
    std::cout << "A's constructor with value: " << x << std::endl;
  }
  ~A() {
    std::cout << "A's deconstructor" << std::endl;
  }
  virtual void foo() {
      std::cout << "A::foo" << std::endl;
  }
private:
  int a;
};

class B : public A {
public:
  B(int x, int y)
      : A(x) {  // 调用父类 A 的构造函数
    std::cout << "B's constructor with value: " << y << std::endl;
  }
  ~B() {
    std::cout << "B's deconstructor" << std::endl;
  }
  void foo() override {
      std::cout << "B::foo" << std::endl;
  }

  int b;
};

class C : public B {
public:
  C(int x, int y, int z)
      : B(x, y) {  // 调用父类 B 的构造函数
    std::cout << "C's constructor with value: " << z << std::endl;
  }
  ~C() {
    std::cout << "C's deconstructor" << std::endl;
  }
  void foo() override {
      std::cout << "C::foo" << std::endl;
  }
  int c;
};

int main() {
  C obj(1, 2, 3);
  obj.foo();
  return 0;
}
```

输出内存布局为：

```
*** Dumping AST Record Layout
         0 | class A
         0 |   int a
           | [sizeof=4, dsize=4, align=4,
           |  nvsize=4, nvalign=4]

*** Dumping AST Record Layout
         0 | class B
         0 |   class A (base)
         0 |     int a
         4 |   int b
           | [sizeof=8, dsize=8, align=4,
           |  nvsize=8, nvalign=4]

*** Dumping AST Record Layout
         0 | class C
         0 |   class B (base)
         0 |     class A (base)
         0 |       int a
         4 |     int b
         8 |   int c
           | [sizeof=12, dsize=12, align=4,
           |  nvsize=12, nvalign=4]

```

虚函数表布局：

```
Vtable for 'C' (3 entries).
   0 | offset_to_top (0)
   1 | C RTTI
       -- (A, 0) vtable address --
       -- (B, 0) vtable address --
       -- (C, 0) vtable address --
   2 | void C::foo()

VTable indices for 'C' (1 entries).
   0 | void C::foo()

Vtable for 'B' (3 entries).
   0 | offset_to_top (0)
   1 | B RTTI
       -- (A, 0) vtable address --
       -- (B, 0) vtable address --
   2 | void B::foo()

VTable indices for 'B' (1 entries).
   0 | void B::foo()

Vtable for 'A' (3 entries).
   0 | offset_to_top (0)
   1 | A RTTI
       -- (A, 0) vtable address --
   2 | void A::foo()

VTable indices for 'A' (1 entries).
   0 | void A::foo()

```

## C++ 类的成员变量初始化顺序

- 先执行列表初始化，列表初始化未提供的则执行默认初始化
- 再执行相应的构造函数初始化逻辑

```C++
class Complex {
public:
    Complex() {}
    Complex(float re) : re_(re) {}
    Complex(float re, float im) : re_(re) , im_(im) {}
private:
    float re_{0};
    float im_{0};
};
```

## 当父类存在父类, C++ 派生类的构造和析构顺序

对于上述代码C继承B，B继承A，构造一个 C 的输出为：

```
A's constructor with value: 1
B's constructor with value: 2
C's constructor with value: 3
C::foo
C's deconstructor
B's deconstructor
A's deconstructor
```

有点像是父类是子类的成员变量，但继承与组合的区别是：

1. **继承 vs 组合**：
   - **继承**（父类构造函数）：子类继承父类的属性和行为，子类是父类的一种特殊类型。
   - **组合**（成员变量构造函数）：一个类包含另一个类的对象，表示“有一个”的关系。

2. **调用顺序**：
   - **继承**：父类的构造函数在子类的构造函数之前被调用。
   - **组合**：成员对象的构造函数在包含类的构造函数之前被调用。

3. **访问权限**：
   - **继承**：子类可以访问父类的受保护成员（`protected`），但不能访问 私有成员（`private`）。
   - **组合**：包含类只能通过成员对象的公共接口访问其成员。

4. **内存布局**：
   - **继承**：子类对象包含父类对象的内存布局，子类对象的内存布局是父类对象的扩展。
   - **组合**：包含类对象包含成员对象的内存布局，成员对象是包含类对象的一部分。

## 使用父类的构造函数构造自己

以下是一个示例：

```cpp
#include <iostream>

class Base {
public:
    Base(int x) {
        std::cout << "Base constructor called with x = " << x << std::endl;
    }
};

class Derived : public Base {
public:
    using Base::Base; // Inherit constructors from Base
};

int main() {
    Derived d(42); // Calls Base(int x) constructor
    return 0;
}
```

在这个例子中，`Derived`类继承了`Base`类的构造函数，因此可以直接使用`Base`类的构造函数来初始化`Derived`类的对象。运行这个程序将输出：

```
Base constructor called with x = 42
```

在C++中，`using base::base;`这种语法用于继承构造函数。这意味着派生类将继承基类的构造函数，从而可以直接使用基类的构造函数来初始化派生类的对象。

## C++ 类和结构体的区别

除了 struct 缺省的成员属性是 public，而 class 是 private 外没有区别。

