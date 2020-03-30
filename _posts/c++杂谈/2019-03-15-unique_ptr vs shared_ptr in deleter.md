---
?layout: post
title: deleter in smart-pointer
category: 编程语言
tags: [c++, unique_ptr, shared_ptr]
keywords: C++, unique_ptr, shared_ptr, deleter
---
---

## unique_ptr声明
``` cpp
template<
    class T,
    class Deleter = std::default_delete<T>
> class unique_ptr;
template <
    class T,
    class Deleter
> class unique_ptr<T[], Deleter>;
```

## shared_ptr声明
``` cpp
template< class T > class shared_ptr;
```

## 两者的区别
- unique_ptr在编译时绑定删除器，所以改变deleter对于unique_ptr是一件困难的事情。
- shared_ptr在运行时绑定删除器，且shared_ptr不是将deleter保存为一个成员，因为删除器的类型直到运行时才知道。

## 副作用
``` cpp
template<class Deleter, class T>
Deleter& unique_ptr<T, Deleter>::get_deleter();

template<class Deleter, class T>
Deleter* get_deleter(const std::shared_ptr<T>& p);
```
- get_deleter可以作为unique_ptr的成员函数
- get_deleter不能作为shared_ptr的成员函数

## 使用上的区别
``` cpp
class Base {
  public:
  	Base() {std::cout << "Base()" << std::endl;}
    ~Base() {std::cout << "~Base()" << std::endl;}
};

class Derived : public Base {
  public:
  	Derived() {std::cout << "Derived()" << std::endl;}
  	~Derived() {std::cout << "~Derived()" << std::endl}
};

void fun1() {
    std::shared_ptr<Base> sb = std::make_shared<Derived>();
}

void func2() {
    std::unique_ptr<Base> sb = std::make_shared<Derived>();
}

int main()
{
    func1();
    func2();
}
```
**输出**
- func1()
  - Base()
  - Derived()
  - ~Derived()
  - ~Base()
- func2()
  - Base()
  - Derived()
  - ~Base()

## 原因

- unique_ptr的目的仅仅只是为了高效的管理指针，所以其deleter作为成员变量，这样在调用的时方便快捷，所以func2中直接将Base的析构函数作为了deleter
- shared_ptr则是为了共享raw pointer，在它的control block中必然包含了pointer和refcount，只有当refcount为0的时候才调用deleter，所以shared_ptr的设计哲学是：既然调用次数不是很多那就不将deleter作为member了，所以在shared_ptr的refcount为0是，可能会发生：

``` cpp
del ? del(p) : delete p;
```
上述代码的意思是：如果存在删除器，则调用删除器完成p的析构，否则直接delete p，在func1中，p为Derived类型


## 参考

[1] [stackoverflow deleter-type-in-unique-ptr-vs-shared-ptr](https://stackoverflow.com/questions/27742290/deleter-type-in-unique-ptr-vs-shared-ptr)

[2] [cppreference-unique-ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)

[3] [cppreference-shared-ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)

[4] c++ primer 第五版

