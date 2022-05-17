---
layout: post
title: "如何使用C++ unique_ptr - 来自Rust所有权机制的启发"
date: 2022-05-15 12:50:09 +0800
categories: 原创
---

## Rust所有权机制

Rust的所有权机制（Ownership）是一种独特的管理“值”的生命周期的方式。其基本规则是：

1. Each value in Rust has a variable that’s called its owner. Rust里每一个值都对应一个变量，称作其所有者。
2. There can only be one owner at a time. 同一时刻只能有一个所有者。
3. When the owner goes out of scope, the value will be dropped. 当所有者离开作用域的时候，值就会被销毁。

在Rust中，`=`赋值和函数传参，都会隐含地导致，“值”的所有权发生转移。因此，

```
let s1 = String::from("hello");
do_somthing_with(s1);

println!("{}, world!", s1);  // 此时，s1不再指向"hello"字符串，所以会编译报错
```

为了避免函数调用造成“值”的所有权发生变成更，Rust引入了一个概念，叫做“引用”。引用有点像指针，但是编译器保证引用所指向的地址一定不为空。

```
let s1 = String::from("hello");
do_somthing_with(&s1); // 使用引用传递变量

println!("{}, world!", s1);  // 此时，s1依然指向"hello"字符串，可以正常使用
```

Rust的引用默认为不可变，如果想要修改引用所指向的“值”，需要显式的声明：

```
do_somthing_with(mut &s1);  
```

为了避免一个值被多个函数同时修改，Rust限制同一时刻只能有一个可变引用，不可变引用则没有数量限制。

### 小结

Rust中可以有两种方式接触到值：
1. 成为其所有者；
2. 使用引用。

从代码形式来看，函数声明可以有三种方式，分别表示：
1. 要成为所有者，掌控值的生命周期
2. 只想取得值的内容
3. 想要修改值的内容

```
struct Object {
    content: String,
}
fn control_lifecycle(o: Object) -> Object {};
fn only_check_value(o: &Object) {};
fn modify_it_content(o: mut &Object) {};
```

## C++的智能指针

为了科学管理对象生命周期，现代C++引入了一组智能指针来管理对象指针。

* `shared_ptr`，当对象指针需要被共享的时候使用。
* `unique_ptr`，当对象指针无需被共享的时候使用。
* `weak_ptr`，和`shared_ptr`配套使用，不掌控对象生命周期。

`shared_ptr`在函数传递，复制，离开作用域等时刻，会原子的更新引用计数。而且，除了指针本身，`shared_ptr`对象还需要另外分配指针，指向控制区（`control block`）。也就是一个shared_ptr对象，需要两个指针的空间。C++通常都使用在性能敏感的场景，因此运行时开销较大的`shared_ptr`往往需要避免使用。

`unique_ptr`不存在上述运行时开销，性能较为理想。不过，函数传递`unique_ptr`的时候，无从确保其指向的对象不被修改。而`const std::unique<X> x`只能确保x本身不能被转移所有权。

### Rust的启发

根据前面的讨论，滥用`unique_ptr`会丢失编译期的对变量修改的检查。如果要保留这些能力，还得需要const和引用的参与。因此，类似于前面跟据函数作用对Rust函数的分类，C++里也可以有相近的做法。

```
class Object {
public:
    explicit Object(std::string s): content(std::move(s)) {};
    std::string content; 
};

std::unique_ptr<Object> control_lifecycle(std::unique_ptr<Object> o) {
    o->content = "owner changed it";
    return o;
};

void only_check_value(const Object& o) {
    std::cout << "It contains: " << o.content << std::endl;
};

void modify_it_content(Object& o) {
    o.content = "3rd-party changed it";
}; 

int main() {
    auto o = std::make_unique<Object>("Hello");
    only_check_value(*o); // It contains: Hello

    o = control_lifecycle(std::move(o)); // ownership move out and then in
    only_check_value(*o); // It contains: owner changed it
    
    modify_it_content(*o);
    only_check_value(*o); // It contains: 3rd-party changed it

    return 0;
}
```

这种写法，好处是从签名就能看出来函数的作用。缺点是，跟`shared_ptr`相比，需要程序员**手动**确保const引用在使用期间，unique_ptr对象不会被析构。

## 参考资料

* [Understanding Ownership - Rust, the Book](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
* [std::unique_ptr - cppreference.com](https://en.cppreference.com/w/cpp/memory/unique_ptr)
