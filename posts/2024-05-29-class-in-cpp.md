---
title: C++ 的类
categories: Dev
tags: [Linux, C++, Class] 
created: 2024-5-29 23:18:00
---

最近经常能够看到 C++ 面经中出现 C++ 类的内存布局及初始化一个派生类时，基类，派生类的构造顺序等这样的问题。同时自己也在这方面存在疑惑，所以就专门去学习了相关的知识，并在查找资历的过程中记录一下。

在开始之前，先看一段代码的输出：

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
  int x_{0};
};

class B : virtual public A {
public:
  B(int x, int y)
      : A(x) {  // 调用父类 A 的构造函数，由于显示的定义了参数构造函数，不会生成默认构造函数
    std::cout << "B's constructor with value: " << y << std::endl;
  }
  ~B() {
    std::cout << "B's deconstructor" << std::endl;
  }
  void foo() override {
    std::cout << "B::foo" << std::endl;
  }

  int x_{1};
};

class C : virtual public A {
public:
  C(int x, int y)
      : A(x) {  // 调用父类 A 的构造函数
    std::cout << "C's constructor with value: " << y << std::endl;
  }
  ~C() {
    std::cout << "C's deconstructor" << std::endl;
  }
  void foo() override {
    std::cout << "C::foo" << std::endl;
  }
  void bar() {
    std::cout << "C::bar" << std::endl;
  }
  int x_{2};
};

class D : public B, public C {
public:
  D(int x, int y, int z)
      :A(x), B(x, y), C(y, x) {  // 调用父类 B 的构造函数，这里必须调用A的参数构造函数
    std::cout << "D's constructor with value: " << z << std::endl;
  }
  ~D() {
    std::cout << "D's deconstructor" << std::endl;
  }
  void foo() override {
    std::cout << "D::foo" << std::endl;
  }
  void bar() {
    std::cout << "D::bar" << std::endl;
    std::cout << "D::x = " << " " << x_ << std::endl;
    std::cout << "C::x = " << " " << C::x_ << std::endl;
    std::cout << "B::x = " << " " << B::x_ << std::endl;
    C::bar();
  }
  int x_{3};
};

int main() {
  D d(1, 2, 3);
  std::cout << "\n";

  d.bar();
  std::cout << "\n";

  C* c = &d;
  c->bar();
  c->foo();
  std::cout << "\n";

  return 0;
}
```

output:

```shell
A's constructor with value: 1
B's constructor with value: 2
C's constructor with value: 1
D's constructor with value: 3

D::bar
D::x =  3
C::x =  2
B::x =  1
C::bar

C::bar
D::foo

D's deconstructor
C's deconstructor
B's deconstructor
A's deconstructor
```

## C++ 类内存布局

执行

```bash
clang++ -Xclang -fdump-record-layouts xxx.cpp
```

输出内存布局为：

```

*** Dumping AST Record Layout
         0 | class A
         0 |   (A vtable pointer)
         8 |   int x_
           | [sizeof=16, dsize=12, align=8,
           |  nvsize=12, nvalign=8]

*** Dumping AST Record Layout
         0 | class B
         0 |   (B vtable pointer)
         8 |   int x_
        16 |   class A (virtual base)
        16 |     (A vtable pointer)
        24 |     int x_
           | [sizeof=32, dsize=28, align=8,
           |  nvsize=12, nvalign=8]

*** Dumping AST Record Layout
         0 | class C
         0 |   (C vtable pointer)
         8 |   int x_
        16 |   class A (virtual base)
        16 |     (A vtable pointer)
        24 |     int x_
           | [sizeof=32, dsize=28, align=8,
           |  nvsize=12, nvalign=8]

*** Dumping AST Record Layout
         0 | class D
         0 |   class B (primary base)
         0 |     (B vtable pointer)
         8 |     int x_
        16 |   class C (base)
        16 |     (C vtable pointer)
        24 |     int x_
        28 |   int x_
        32 |   class A (virtual base)
        32 |     (A vtable pointer)
        40 |     int x_
           | [sizeof=48, dsize=44, align=8,
           |  nvsize=32, nvalign=8]
```


执行

```shell
clang++ -Xclang -fdump-vtable-layouts xxx.cpp
```

查看虚函数表实体

虚继承的作用:

- **消除二义性**：在多重继承中，避免了对同一基类的多重实例。
- **共享基类数据**：派生类可以共享基类的成员，节省内存。
- **统一接口**：确保派生类通过虚基类访问基类的接口一致。

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

先构造的后析构

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

