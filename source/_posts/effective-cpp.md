---
title: Effective C++ 读书笔记
tags: [C++]
categories: 读书笔记
date: 2020-09-26 22:18:51
---

# 1. 让自己习惯C++

## 条款01：视C++为一个语言联邦

C++有4种打开方式，不同方式的最佳实践也不同，需要适时切换：
1. C。就是C啦……
2. Object-Oriented C++。面向对象的基本概念：类/继承/虚函数/多态……
3. Template C++。即泛型编程，还包括模板元编程。
4. STL。虽然STL是一个template库，但它有自己的使用方式。

<!-- more -->

## 条款02：尽量以const, enum, inline替换#define

### 1. 拒绝#define常量

宏定义的常量不会进入符号表，可能导致追踪困难。此外还会使该值有多份拷贝。
```cpp
// bad
#define ASPECT_RATIO 1.653

// good
const double AspectRatio = 1.653;
```

### 2. 拒绝#define函数

宏定义的函数调用需要很多括号，而且容易写错。
```cpp
// ugly & wrong
// 如果a是x++的话，当a>b时，x将+2，否则x+1
#define CALL_F_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))

// good
template<typename T>
inline void callFWithMax(const T& a, const T& b) {
    f(a > b ? a : b);
}
```

### 3. enum hack

对于一个足够现代的编译器，直接在`static int/char/bool...`等整形成员的声明中就赋值是合法的，也就是：

```cpp
class MyClass {
private:
    static const int Size = 5;
    int array_[Size];
}
```

但如果编译器对这种写法不开心，非得让你把声明和定义分开：

```cpp
class MyClass {
private:
    static const int Size;
    int array_[Size];
}

const int MyClass::Size = 5;
```

那么`array_`就会编译报错……这时可以通过`enum hack`来搞定
```cpp
class MyClass {
private:
    enum { Size = 5 };
    int arrar_[Size];
}
```

## 条款03：尽可能使用const

`const`可以用在很多地方。不过曾经看到有人说`const`这个关键字起名起的不好，应该叫`read_only`，`constexpr`才是`const`，我表示赞同。于是本小节的例子用`read_only`代替`const`，应该能方便理解。

### 1. read_only变量

```cpp
#define read_only const

char hello[] = "hello";
char* p1 = hello; // p1无约束
read_only char* p2 = hello; // p2指向"hello"，不能修改"hello"
p2 = "world"; // 但可以修改p2的指向
char* read_only p3 = hello; // p3是read_only的，即指向固定，但可以改"hello"的值
read_only char* read_only p4 = hello; // p4是read_only的，指向固定，指向的数据也是read_only的
```

### 2. read_only成员函数

为什么需要`read_only`成员函数

1. 使得使用者清楚地知道哪些函数不会修改对象
2. 使得`read_only`的对象有用武之地

第1点很好理解，使用标记为`read_only`的对象天然的有一种安全感……第2点是怎么回事呢？

假设我有这样一个类
```cpp
#include <string>
#define read_only const
using namespace std;

class MyClass {
public:
    explicit MyClass(read_only string &s) { value = s; }; // 拷贝赋值

    char &GetChar(size_t pos) { return value[pos]; }

private:
    string value;
};
```

那么自然这样的操作是没问题的
```cpp
int main() {
    MyClass obj("hello world");
    obj.GetChar(0) = 'i'; // 显然是合法的
    return 0;
}
```

但如果我声明了一个`read_only`的对象

```cpp
int main() {
    read_only MyClass obj("hello world");
    obj.GetChar(0); // 报错，我连读都不能读
    return 0;
}
```

这时候我们就需要加上一个`read_only`的版本

```cpp
...
read_only char &GetChar(size_t pos) read_only { return value[pos]; }
...
```

### 3. Don't Repeat Yourself

可以看到上面`read_only`版本的代码和原版几乎一样，这个例子就一行`return`区别不大，但对于某些逻辑复杂的函数来说就麻烦了。

```cpp
char& GetChar(size_t pos) {
    // check bound
    // log access
    // verify data
    return value[pos];
}

read_only char& GetChar(size_t pos) read_only {
    // check bound
    // log access
    // verify data
    return value[pos];
}
```

这时可以采用casting来避免重复

```cpp
read_only char& GetChar(size_t pos) read_only {
    // check bound
    // log access
    // verify data
    return value[pos];
}

char& GetChar(size_t pos) {
    return const_cast<char&>( // 3. drop掉read_only限制
        static_cast<read_only MyClass&>(*this) // 1. 转为read_only
        .GetValue(pos) // 2. 调用read_only版本的
    );
}
```

注意，一定是非`read_only`版本调用`read_only`版本，而不是反过来。不然就违反了`read_only`的成员函数不可改变对象的承诺。


## 条款04：确定对象使用前已被初始化

很自然的一个条款，有几条具体要求：

1. 因为很难记住C++里哪些地方能保证初始化，所以干脆不记，而是全部自行初始化
2. 在实现构造函数的时候，请使用`initialization list`，也就是
   ```cpp
   // bad
   class MyClass {
       MyClass(const std::string& name, const std::string& address) {
           name_ = name;
           address_ = address;
       }

   // good
   class MyClass {
       MyClass(const std::string& name, const std::string& address)
           : name_(name), address_(address) {}
   }
   ```
   原因是前者会调用默认构造函数+各个成员变量的拷贝赋值函数；而后者仅调用各个成员变量的拷贝构造函数。
3. 当你需要一个跨编译单元的全局变量的时候，请使用函数+`static`变量代替：
   ```cpp
   // bad

   // a.h
   class FileSystem {
       ...
       size_t numDisks() const;
       ...
   }
   extern FileSystem fs;
   // b.h
   class Directory {
       Direcotory(params) {
           ...
           size_t disks = fs.numDisks();
           ...
       }
   }
   // c.cpp
   Directory temp_dir(params);
   ```
   如果`temp_dir`先初始化，那么程序就挂了，因为`fs`还没初始化。但下面的写法就没有问题。

   ```cpp
   // good

   // a.h
   class FileSystem {
       ...
       size_t numDisks() const;
       ...
   }
   inline FileSystem& GetFileSystem() {
       static FileSystem fs;
       return fs;
   }
   // b.h
   class Directory {
       Direcotory(params) {
           ...
           size_t disks = GetFileSystem().numDisks();
           ...
       }
   }
   // c.cpp
   Directory temp_dir(params);
   ```