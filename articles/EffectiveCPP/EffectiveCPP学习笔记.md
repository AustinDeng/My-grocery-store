---
title: 《Effective C++》学习笔记
date: 2021-05-28 18:53:13
tags: 
- C++
- 学习笔记
categories: 
- C++
---

## 让自己习惯C++

### 条款01： 视 C++ 为一个语言联邦

C++ 包含了多种泛型编程：
- 过程形式
- 面向对象形式
- 函数形式
- 泛型形式
- 元编程形式

将C++视作一个由相关语言组成的联邦而非单一语言：
- C。
    - 区块、语句、预处理器、内置数据类型、数组、指针
    - 没有模板、没有异常、没有重载
- Object-Oriented C++。
    - 类、封装、继承、多态、virtual函数(动态绑定)
- Template C++：
    - 泛型编程
    - template metaprogramming(TMP 模板元编程)
- STL

C++ 高效编程守则视状况而变化，取决于你使用C++哪一部分。

<!-- more -->

### 条款02： 尽量以 const，enum，inline 替换 #define

将 `#define ASPECT_RATIO 1.653`替换为`const double AspectRatio = 1.653`

定义一个常量的`char*-based`字符串
将`const char* const authorName = "Scott Meyers";`替换为`const std::string authorName("Scott Meyers");`

我们无法利用`#define`创建一个 `class`专属常量，因为`#define`并不重视作用域(scope)

#### emum hack

```C++
class GamePlayer {
private:
    static const int NumTurn = 5;
    int scores[NumTurn];
    ...
}
```
由于编译器坚持必须在编译期间知道数组大小，可能导致编译器(错误地)不允许“static 整数型 class 常量”完成“in class 初值设定”

所以可以这么做
```C++
class GamePlayer {
private:
    enum { NumTurn = 5 };
    int scores[NumTurn];
    ...
}
```

enum hack 的行为某方面更像#define而不像const。
例如：取一个const的地址是合法的，但取一个enum的地址就不合法，而取一个#define的地址也不合法。

### 使用`#define`实现宏(macros)

```C++
// 以 a 和 b 的较大值调用f
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

会遇到这样的问题

```C++
int a = 5, b = 0;
CALL_WITH_MAX(++a, b);     // a 被累加二次
CALL_WITH_MAX(++a, b+10);  // a 被累加一次
```

只要用 template inline 解决即可

```C++
template<typename T>
inline void callWithMax(const T& a, const T& b) {    // 由于不知道T是什么，所以采用 pass by reference-to-const
    f(a > b ? a: b);
}
```

### 总结
- 对于单纯常量，最好以 const 对象或 enums 替换 #define
- 对于形似函数的宏(macros),最好改用inline函数替换 #defines

### 条款03： 尽可能使用 const

面对指针，你可以指出指针自身、指针所指物，或者两者都（或都不）是const

```C++
char greeting[] = "Hello";         
char* p = greeting;                // non-const pointer, non-const data
const char* p = greeting;          // non-const pointer, const data
char* const p = greeting;          // const pointer, non-const data
const char* const p = greeting;    // const pointer, const data
```
如果关键字 const 出现在星号左边，说明被指物是常量
如果关键字 const 出现在星号右边，说明指针自身是常量
如果关键字 const 出现在星号两边，说明被指物和指针两者都是常量

对于一下函数参数得写法虽然不同，但是意义是一样得
```C++
void f1(const Widget* pw);
void f2(Widget const * pw);
```

在 STL 中，如果希望迭代器所指的东西不可被改动， 可以使用 const_iterator

令函数返回一个常量值，往往可以降低因客户错误而造成的意外，而又不至于放弃安全性和高效性。
```C++
class Rational {...}
const Rational operator* (const Rational& lhs, const Rational& rhs);
```
返回一个 const 对象可以阻止这样的暴行：
```C++
Rational a, b, c;
...
(a*b) = c;
```

### const 成员函数
将 const 实施于成员函数的目的，是为了确认该成员函数可用作于 const 对象身上。

两个成员函数如果只是常量性(constness)不同，可以被重载。
```C++
class TextBlock {
public:
    ...
    const char& operator[](std::size_t position) const {
        return text[position];
    }

    char& operator[](std::size_t position) {
        return text[position];
    }
private:
    std::string text;
}
```

成员函数如果是 const 意味着什么？
有两个流行的概念去解释： bitwise constness 以及 logical constness

### bitwise const
成员函数只有在不更改对象的任何成员变量时才可以是const。
不幸的是，许多成员函数虽然不十足具备 const 性质却能通过 bitwise 测试。

```C++
class CTextBlock {
public:
    ...
    char& operator[] (std::size_t position) const   // bitwise const 声明
    { return pText[position]; }                     // 但其实不当
private:
    char* pText;
};

const CTextBlock cctb("Hello");    // 声明一个常量对象
char* pc = &cctb[0];               // 调用const operator[]取得一个指针，指向cctb的数据

*pc = 'J';                         // cctb 现在变成了"Jello"
```

### logical constness

一个 const 成员函数可以修改它所处理的对象内的某些 bits，但只有在客户端侦测不出的情况下才得如此。

```C++
class CTextBlock {
public:
    ...
    size_t length() const;
private:
    char* pText;
    size_t textLength;      // 最近一次计算的文本区块长度
    bool lengthIsValid;     // 目前的长度是否有效
};

size_t CTextBlock::length() const {
    if(!lengthIsValid) {
        textLength = strlen(pText);     // 错误！在 const 成员函数内不能赋值给 textLength 和 lengthIsValid
        lengthIsValid = true;
    }
    return textLength;
}
```

可以利用与 const 相关的摆动场： mutable(可变的)。
mutable 释放掉 non-static 成员变量的 bitwise constness 约束。

```C++
class CTextBlock {
public:
    ...
    size_t length() const;
private:
    char* pText;
    mutable size_t textLength;      // 这些成员变量可能总是会被更改
    mutable bool lengthIsValid;     // 即使是在 const 成员函数内
};

size_t CTextBlock::length() const {
    if(!lengthIsValid) {
        textLength = strlen(pText);     // 此时编译器就不会报错了
        lengthIsValid = true;
    }
    return textLength;
}
```

### 在 const 和 non-const 成员函数中避免重复

```C++
class TextBlock {
public:
    ...
    const char& operator[] (size_t position) const {
        ... // 边界检验(bounds checking)
        ... // 志记数据访问(log access data)
        ... // 校验数据完整性(verify data integrity)
        return text[position];
    }
    const char& operator[] (size_t position) {
        ... // 边界检验(bounds checking)
        ... // 志记数据访问(log access data)
        ... // 校验数据完整性(verify data integrity)
        return text[position];
    }
private:
    string text;
};
```

显然上面的代码出现了非常多的冗余，因此我们可以在 non-const operator[] 中调用 const operator[]
(思考：可以在const operator[] 中调用 non-const operator[]吗？)

```C++
class TextBlock {
public:
    ...
    const char& operator[] (size_t position) const {
        ... // 边界检验(bounds checking)
        ... // 志记数据访问(log access data)
        ... // 校验数据完整性(verify data integrity)
        return text[position];
    }
    const char& operator[] (size_t position) {
        return 
            const_cast<char&>(                                      // 将 op[] 返回值的const转除
                static_cast<const TextBlock&>(*this)                // 为 *this 加上 const
                    [position]                                      // 调用 const op[]
            );
    }
private:
    string text;
};
```

### 总结：
- 将某些东西声明为 const 可帮助编译器侦测出错误用法。const 可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。
- 编译器强制实施 bitwise constness，但编写程序时应当使用“概念上的常量性”(conceptual constness)
- 当 const 和 non-const 成员函数有着实质等价的实现时，令 non-const 版本调用const版本可以避免代码重复。


### 条款04： 确定对象被使用前已被初始化

- 永远在使用对象之前先将它初始化。
- 确保每一个构造函数都将对象的每一个成员初始化。

构造函数的一个比较好的写法就是：使用成员初始列(member initialization list)替换赋值动作。

为避免某些可能存在的晦涩的错误，当成员初值列中条列各个成员时，最好总是以其声明次序为次序。
这里晦涩错误是指，两个成员变量的初始化带有次序性。比如初始化array时要指定大小，因此代表大小的那个成员变量必须先有初值。

### 不同编译单元内定义的 non-local static 对象的初始化次序

假设有一个FileSystem class，它让互联网上的文件看起来好像位于本机(local)。
由于这个class使世界看起来像一个单一文件系统，所以需要一个特殊对象，位于global或者namespace作用域内，象征单一文件系统。

```C++
class FileSystem {              // 位于你的程序库
public:
    ...
    size_t numDisks() const;    // 众多成员函数之一
    ...
};
extern FileSystem tfs;          // 预备给客户使用的对象， tfs 表示“the file system”
```

现在假设某些客户建立了一个class用以处理文件系统内的目录(directories)，很自然他们的class会用上 theFileSystem 对象：
```C++
class Directory {               // 由程序库客户建立
public:
    Directory(params);
    ...
};

Directory::Directory(params) {
    ...
    size_t disks = tfs.numDisks();   // 使用tfs对象
    ...
}

Directory tempDir(params);      // 为临时文件而创建的目录

```

此处 tfs 要在tempDir之前被初始化。但是tfs和tempDir是不同的人在不同时间于不同的源码文件建立起来的，它们是定义于不同编译单元内的non-local static 对象。

正确的做法：将每一个 non-local static 对象搬到自己的专属函数内。
这些函数返回一个reference指向它所含的对象。然后用户直接调用这个函数，而不直接指涉这些对象。

```C++
class FileSystem {...};            // 同上
FileSystem& tfs() {                // 这个函数用来替换 tfs 对象： 它在FileSystem class 中可能是个static。
    static FileSystem fs;          // 定义并初始化一个 local static 对象。
    return fs;                     // 返回一个 reference 指向上述对象。
}
class Directory {...};
Director::Directory(params) {
    ...
    size_t disk = tfs().numDisks();     // 原来是reference to tfs，现在是tfs()
    ...
}
Directory& tempDir() {          // 这个函数用来替换 tempDir 对象，他在Directory class 中可能是一个static
    static Directory td;        // 定义并初始化一个 local static 对象
    return td;                  // 返回一个reference指向上述对象
}

```

### 总结：
- 为内置类型对象进行手工初始化，因为c++不保证初始化它们。
- 构造函数最好使用成员初值列(member initialization list),而不要在构造函数中使用赋值操作(assignment)。初值列列出的成员变量，其排列次序应该和他们的声明次序相同。
- 为免除“跨编译单元的初始化次序”问题，请以local static 对象替换non-local static 对象。

## 构造/析构/赋值运算

### 条款05： 了解 C++ 默默编写并调用哪些函数

### 条款06： 若不想使用编译器自动生成函数，就该明确拒绝

### 条款07： 为多态基类声明 virtual 析构函数

### 条款08： 别让异常逃离析构函数

### 条款09： 绝不在构造和析构过程中调用 virtual 函数

### 条款10： 令 operator= 返回一个 reference to *this

### 条款11： 在 operator= 中处理“自我赋值”

### 条款12： 复制对象时勿忘其每一个成分

## 资源管理

## 设计与声明

## 实现

## 继承与面向对象设计

## 模板与泛型编程

## 定制 new 和 delete

## 杂项讨论
