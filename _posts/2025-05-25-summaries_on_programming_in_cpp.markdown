---
layout: post
title: "对C++编程的一些总结"
date: 2025-05-25 11:22:52 +0800
categories: 原创
---

> 前几天，写了一段时间的C++。我发现，每次写C++都像从头学习，即使有了AI，很多东西都得再查一边。所以说，该沉淀的知识还是得记下来。

## 关于const

关键字const可以：
- 修饰变量定义或者方法入参，让其不可修改。
- 修饰方法返回值，让其返回值不可修改。

关键词const还可以修饰方法本身。在const方法里，this指针为不可变。也就是，不能修改对象的成员变量。

同时，同一个方法定义，可以有const版本和非const版本。如果两个版本同时存在，const对象只会调用const版本方法，而非const对象只会调用非const版本方法。

如果只有一个版本，那么:
- 非const对象可以调用任意版本方法
- const对象无法调用非const版本方法

所以，**应该尽可能将方法定义为const版本**。

```c++
#include <iostream>

class Foo {
public:
    void print() {
        i++;
        std::cout << "non-const version" << std::endl;
    }
    void print() const {
        std::cout << "const version" << std::endl;
    }
private:
    int i = 0;
};

int main() {
    Foo a;
    a.print(); // 打印得到"non-const version"
    const Foo b;
    b.print(); // 打印得到"const version"
}
```

## 尽量给构造器增加explicit关键字

C++里，单参数构造器默认可以隐式转换类型。这样，一个类型为T的函数参数，就可以接受一个类型为X的入参，只要类T有定义一个入参类型为X的单参数构造器。

这种特性，有时会有用。例如，RocksDB的Slice类。在需要Slice类型参数的地方，传入一个`const std::string&`可以直接使用，避免了临时构造Slice对象的模板式代码，更方便。

有的时候，这种类型转换也许是不需要的。那就在构造器的加上关键字explicit，就可以避免该构造器参与隐式类型转换。此时，该构造器必须显式的调用。

多参数构造器，虽然没有隐式类型转换的问题。但explicit关键字同样有用。因为可以避免滥用复制列表初始化，造成“先构造再复制”。

例如，下面这个例子中，vector的push_back方法会先执行列表初始化构造对象，然后复制对象到vector内部的内存上。如果构造器上标记有explicit关键字，则会编译报错。如果使用emplace_back方法，则不会报错。因为内存分配发生在vector的内部，原地构造对象，不会发生复制行为。

```c++
class Foo {
...
    explicit Foo(int a, string b) : a(a), b(b) {}
...
};
vector<Foo> v;
v.push_back({1, "a"});  // 编译报错
v.emplace_back(1, "a"); // 编译通过
```

## 显式删除不需要的构造器

C++允许定义复制构造器，复制赋值构造器，移动构造器，移动赋值构造器。这些可以帮助C++程序员自定义，对象在复制和移动时候的行为。

不过对于非值对象，这类方法往往没什么用，还会因为意外的隐式调用产生错误。一个好的方法是，将这些方法都标记为删除，不允许编译器自动生成对应方法。这样，当意外的调用发生时，编译器会报错提醒我们。

```c++
class Foo {
public:
    Foo(const Foo&) = delete;
    Foo& operator=(const Foo&) = delete;
    Foo(Foo&&) = delete;
    Foo& operator=(Foo&&) = delete;
};
```

## 性能敏感的方法可以标记为noexcept

对于性能敏感的方法，可以标记为`noexcept`，表示该方法不会抛出异常。这样编译器可以优化生成的指令，提高性能。

如果异常不可避免，就将方法标记为`noexcept(false)`，提醒此处需要潜在的优化。

## 关于Virtual函数

在父类对象中调用虚函数时，运行时需要查询虚函数表，确定子类的虚函数地址，然后加载指令，执行调用。这个过程存在一定的性能损失：不是因为动态查表，主要是因为虚函数表的存在而无法内联函数。

一种办法是使用奇异递归模板模式CRTP（Curiously Recurring Template Pattern）。

```c++
template<class T>
class Parent {
    void foo() {
        static_cast<T*>(this)->fooImpl();
    }
};

class Child : public Parent<Child> {
    void fooImpl() { ... }
};
```

在这种方式中，父类已经知道了子类的类型。因此，在foo方法中，不再需要查表定位子类的函数定义，而是直接调用，从而为函数内联扫除了障碍。

## 关于Optional

C++ `std::optional`是一个来源于函数式编程范式的Monad，用于表示一个对象可能不存在。

一向声称“零开销抽象”的C++，在这里其实并不是“零开销”。因为`std::optional`的实现，需要一个额外的bool变量来表示对象是否存在，这通常是1个字节。而各平台里，内存地址通常都是对齐的。例如在AMD64 Linux环境里，内存地址按8字节对齐，从而导致对象占用更多的内存空间。

```c++
#include <iostream>
#include <optional>

class Foo {
public:
    Foo() = default;
    void print() const {
        std::cout
            << "size of i: " << sizeof(this->i) << "\n"
            << "size of l: " << sizeof(this->l) << std::endl;
    }
private:
    long i = 0;
    std::optional<long> l = std::nullopt;
};

int main() {
    Foo a;
    std::optional<Foo> b;
    std::optional<std::optional<Foo>> c;

    a.print();
    std::cout << "size of a: " << sizeof(a) << std::endl;
    std::cout << "size of b: " << sizeof(b) << std::endl;
    std::cout << "size of c: " << sizeof(c) << std::endl;
}
```

打印得到

```
size of i: 8
size of l: 16
size of a: 24
size of b: 32
size of c: 40
```

对性能敏感的代码，应该谨慎使用`std::optional`。

