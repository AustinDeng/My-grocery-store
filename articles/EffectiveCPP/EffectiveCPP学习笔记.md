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

#### 使用`#define`实现宏(macros)

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

#### 总结
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

#### const 成员函数
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

#### bitwise const
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

#### logical constness

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

#### 在 const 和 non-const 成员函数中避免重复

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

#### 总结：
- 将某些东西声明为 const 可帮助编译器侦测出错误用法。const 可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。
- 编译器强制实施 bitwise constness，但编写程序时应当使用“概念上的常量性”(conceptual constness)
- 当 const 和 non-const 成员函数有着实质等价的实现时，令 non-const 版本调用const版本可以避免代码重复。


### 条款04： 确定对象被使用前已被初始化

- 永远在使用对象之前先将它初始化。
- 确保每一个构造函数都将对象的每一个成员初始化。

构造函数的一个比较好的写法就是：使用成员初始列(member initialization list)替换赋值动作。

为避免某些可能存在的晦涩的错误，当成员初值列中条列各个成员时，最好总是以其声明次序为次序。
这里晦涩错误是指，两个成员变量的初始化带有次序性。比如初始化array时要指定大小，因此代表大小的那个成员变量必须先有初值。

#### 不同编译单元内定义的 non-local static 对象的初始化次序

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

#### 总结：
- 为内置类型对象进行手工初始化，因为c++不保证初始化它们。
- 构造函数最好使用成员初值列(member initialization list),而不要在构造函数中使用赋值操作(assignment)。初值列列出的成员变量，其排列次序应该和他们的声明次序相同。
- 为免除“跨编译单元的初始化次序”问题，请以local static 对象替换non-local static 对象。

## 构造/析构/赋值运算

### 条款05： 了解 C++ 默默编写并调用哪些函数

当你声明一个空类时，C++的编译器会默认给你声明一个 copy 构造函数、一个copy assignment 操作符以及一个析构函数
若没有声明任何构造函数，编译器也会声明一个 default 构造函数

```C++
class Empty { };
```

上述代码等同于下述代码

```C++
class Empty {
public:
    Empty() { ... }                                 // default构造函数
    Empty(const Empty& rhs) { ... }                 // copy构造函数
    ~Empty() { ... }                                // 析构函数

    Empty& operator= (const Empty& rhs) { ... }     // copy assignment操作符
};
```

思考如下代码：

```C++
template<class T>
class NameObject {
public:
    // 注意这里的 name 是一个 reference-to-non-const 的string类型
    NameObject(std::string& name, const T& value);
    ...   // 此处没有声明 operator=
private:
    // reference
    std::string& nameValue;
    // const
    const T objectValue;
}
```

当执行下面这个代码时，会发生一些什么问题？

```C++
std::string newDog("Peter");
std::string oldDog("Satch");

nameObject<int> p(newDog, 2);
nameObject<int> s(oldDog, 36);

p = s;
```

- p.nameValue 和 s.nameValue 指向不同的 string 对象
- 执行赋值后，p.nameValue 会指向 s.nameValue吗？
- 如果是，则 reference 自身被改动了，但是在C++中，不允许“让 reference 改指向不同对象”
- 若允许“改 reference 指向”, 则会导致修改一个对象，其他reference该对象的都会受到影响

针对以上问题，编译器拒绝回答，即编译到赋值时，拒绝编译！

因此，在C++中：
- 如果打算在一个“内涵reference 成员”的class内支持赋值操作(assignment)，你必须定义 copy assignment操作符
- 同理，对于“内涵 const 成员”也是一样的
- 若某个 base classes 将 copy assignment操作符声明为private，编译器将拒绝为其derived classes 生成一个 copy assignment 操作符

#### 总结：

- 编译器可以暗自为 class 创建 default 构造函数、copy构造函数、copy assignment操作符以及析构函数

### 条款06： 若不想使用编译器自动生成函数，就该明确拒绝

编译器默认创建的函数都是 public 类型的
若不想使用编译器自动生成的函数且不想被调用，可以自己声明并声明类型为private
如此：既阻止了编译器暗自创建其专属版本，又组织了其他人的调用

但是这个方法并不绝对安全，因为 member 函数和friend函数依然可以调用private函数，如果真有此事，连接器会发出错误

可以通过以下方法，将连接期的错误移到编译器：
在一个专门为阻止copying动作而设计的base class 内，将 copy 构造函数和copy assignment操作符声明为private

```C++
class Uncopyable {
protected:              // 允许 derived 对象构造和析构
    Uncopyable() { }
    ~Uncopyable() { }
private:
    Uncopyable (const Uncopyable&);     // 但是阻止拷贝
    Uncopyable& operator=(const Uncopyable&);
};
```

然后在阻止copying的对象中继承这个类即可

```C++
class HomeForSale: private Uncopyable {
    ...
};
```

#### 总结
- 为了驳回编译器自动(暗自)提供的机能，可将相应的成员函数声明为private并且不予实现。
- 使用像 Uncopyable 这样的 base classes 也是一种做法

### 条款07： 为多态基类声明 virtual 析构函数

对于一个多态的基类，如果它的析构函数不是 virtual 的话，会有资源泄漏，败坏数据结构的风险

思考如下关于多态的时钟的例子

```C++
class TimeKeeper {
public:
    TimeKeeper();
    ~TimeKeeper();
    ...
};

class AtomicClock: public TimeKeeper { ... };  // 原子钟
class WaterClock: public TimeKeeper { ... };   // 水钟
class WristWatch: public TimeKeeper { ... };   // 腕表
```

由于客户只想在程序中使用时间，而不操心时间的设计细节
于是可以使用一个工厂(factory)函数，返回指针指向一个计时对象

Factor 函数返回一个base class 指针，指向新生成的derived class 对象

```C++
TimeKeeper* getTimeKeeper();
```

然后，观察下面这个代码

```C++
TimeKeeper* ptk = getTimeKeeper();
...
delete ptk;
```

会产生这样一个问题：
getTimeKeeper 返回的指针指向一个 derived class 对象(例如AtomicClock)
而那个对象却经由一个base 指针(例如一个TimeKeeper*指针)被删除，而目前的base class(TimeKeeper)有个non-virtual函数

C++明确指出：
当derived class 对象经由一个base class指针被删除，而该base class 带着一个non-virtual析构函数，其结果未有定义————实际执行时通常发生的是对象的derived成分没有被销毁(仅仅销毁了base class)

解决方法就是给base class 加上一个virtual析构函数

```C++
class TimeKeeper {
public:
    TimeKeeper();
    virtual ~TimeKeeper();
    ...
};

TimeKeeper* ptk = getTimeKeeper();
...
delete ptk;
```

当class不企图被当作base class时，令其析构函数为virtual往往是一个馊主意。
由于虚函数表机制，这样会导致占用额外的空间，也会导致缺乏移植性
所以，只有当class内含有至少一个virtual函数，才为它声明virtual析构函数

```C++
// 馊主意，因为std::string 有一个non-virtual析构函数
class SpecialString: public std::string {

}

SpecialString* pss = new SpecialString("Impending Doom");
std::string* ps;
...
ps = pss;           // SpecialString* => std::string
...
delete ps;          // 未有定义！现实中*ps的SpecialString资源会泄漏，因为SpecialString析构函数没有被调用
```

#### 纯虚函数
为你希望它成为抽象的那个class声明一个pure virtual 析构函数

#### 总结

- polymorphic(带多态性质的) base classes 应该声明一个virtual析构函数。如果class带有任何virtual函数，它就应该拥有一个virtual析构函数
- Classes 的设计目的如果不是作为base classes使用，或不是为了具有多态性(polymorphically)，就不应该声明 virtual析构函数

### 条款08： 别让异常逃离析构函数

C++不喜欢在析构时吐出异常
考虑一个对象数组，数组中的每一个元素是一个对象，当对象在析构的时候如果产生异常，那么这是一件不确定行为，可能会产生资源泄漏

两种办法解决：
- 如果close抛出异常就结束程序。通常通过abort完成
```C++
DBConn::~DBConn() {
    try {db.close();}
    catch (...) {
        执行运转记录，记录下对close的调用失败;
        std::abort();
    }
}
```
- 吞下因调用close而发生的异常
```C++
DBConn::~DBConn() {
    try {db.close();}
    catch (...) {
        执行运转记录，记录下对close的调用失败;
        std::abort();
    }
}
```

还有一个较好的策略是重新设计DBConn接口，使得客户有机会对可能出现的问题作出反应

#### 总结
- 析构函数绝对不要吐出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吐下它们(不传播)或结束程序。
- 如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数(而非在析构函数中)执行操作

### 条款09： 绝不在构造和析构过程中调用 virtual 函数

不要在构造函数和析构函数期间调用 virtual 函数，因为这样会带来预想不到的结果

假设一个class继承体系，用来表示股市交易，交易有一步是需要审计，所以每当创建一个对象时，需要在审计日志(audit log)中创建一笔适当的记录

```C++
// 所有交易的 base class
class Transaction {
public:
    Transaction();
    // 因类型不同而不同的类型记录，多态性
    virtual void logTransaction() const = 0;

    ...
};
// base class 的构造函数
Transaction::Transaction() {
    ...
    logTransaction();
}

class BuyTransaction: public Transaction {
public:
    virtual void logTransaction() const;
    ...
};

class SellTransaction: public Transaction {
public:
    virtual void logTransaction() const;
    ...
};

```

然后，执行下面这个代码，会发生不符合预期的效果：
```C++
BuyTransaction b;
```

我都知道，构造函数的调用顺序是从内而外的，代码会先调用base class的构造函数，在调用base class构造函数的时候，会调用virtual 的logTransaction.
注意！此时的logTransaction是Transaction内版本，而不是BugTransaction内的版本
但是我们现在是在建立BuyTransaction的对象类型

同样的道理也适用于析构函数。
一旦derived class析构函数开始执行，对象内的derived class成员变量便呈未定义值

一个解决方案是在 class Transaction 内将logTransaction改为non-virtual，然后要求derived class 构造函数传递必要信息给Transaction构造函数，而后那个构造函数便可以安全地调用non-virtual logTransaction。

```C++
class Transaction {
public:
    explicit Transaction(const std::string& logInfo);
    // 如今是一个 non-virtual
    void logTransaction(const std::string& logInfo) const;   

    ...
};

Transaction::Transaction(const std::string& logInfo) {
    ...
    logTransaction(logInfo);
}

class BuyTransaction: public Transaction {
public :
    // 将log信息传递给base class构造函数
    BuyTransaction(parameters) : Transaction(createLogString(parameters)) {
        ...
    }
    ...
private:
    static std::string createLogString(parameters);
};

```

#### 总结
- 在构造和析构期间不要调用virtual函数，因为这类调用从不下降至derived class

### 条款10： 令 operator= 返回一个 reference to *this

为了实现“连锁赋值”，赋值操作符必须返回一个reference指向操作符的左侧实参。
这是你为classes实现赋值操作符时应该遵循的协议

```C++
class Widget {
public:
    ...
    Widget& operator=(const Widget& rhs) {
        ...
        return* this;
    }

    Widget& operator+=(const Widget& rhs) {
        ...
        return* this;
    }

    Widget& operator=(int rhs) {
        ...
        return* this;
    }
    ...
};
```

#### 总结
- 令赋值(assignment)操作符返回一个reference to *this。

### 条款11： 在 operator= 中处理“自我赋值”

自我赋值看起来很愚蠢
```C++
class Widget { ... }
Widget w;
...
w = w;
```

但是有时候潜在的自我赋值却未必能避免

```C++
a[i] = a[j];
*px = *py
```

如果你尝试自行管理资源，可能会掉入“在停止使用资源之前意外释放了它”的陷阱”。

考虑创建一个class用来保存一个指针指向一块动态分配的位图(bitmap)

```C++
class Bitmap { ... };
class Widget {
    ...
private:
    // 指针，指向一个从 heap 分配而得的对象
    Bitmap* pb;
};

// operator= 实现代码，看起来合理，但是自我赋值出现时并不安全
Widget& Widget::operator= (const Widget& rhs) {
    // 停止使用当前的bitmap
    delete pb;
    // 使用rhs’s bitmap 的副本
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

此处的问题是，当自我赋值产生时，operator= 函数内的*this 和rhs有可能是同一对象
那么delete就不只销毁当前对象的bitmap，也销毁了rhs的bitmap

传统方法是增加一个“证同测试”

```C++
Widget& Widget::operator= (const Widget& rhs) {
    // 证同测试
    if(this == &rhs) return *this;

    // 停止使用当前的bitmap
    delete pb;
    // 使用rhs’s bitmap 的副本
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

但是，如果“new bitmap”导致异常(内存不足或者Bitmap copy构造函数抛出异常)
Widget最终会持有一个指针指向一块被删除的Bitmap
这样的指针是有害的，我们无法安全地删除它们，甚至无法安全地读取它们。

```C++
Widget& Widget::operator= (const Widget& rhs) {
    // 记住原先的pb
    Bitmap* pOrig = pb;
    // 令pb指向*pb的一个复制
    pb = new Bitmap(*rhs.pb);
    // 删除原先的pb
    delete pOrig;
    return *this;
}
```

现在，如果“new Bitmap”抛出异常，pb保持原状。及时没有证同测试，这段代码还是可以处理自我赋值

#### 总结
- 确保当对象自我赋值时operator= 有良好的行为。其中技术包括“来源对象”和“目标对象”的地址、精心周到的语句顺序以及copy-and-swap。
- 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。

### 条款12： 复制对象时勿忘其每一个成分

考虑一个class用来表现客户，其中手工设计实现一个copying函数(而不是由编译器创建)，使得外界对它们的调用会被记录(logged)下来。

```C++
// log entry
void logCall(const std::string& funcName);

class Customer {
public:
    ...
    Customer(const Customer& rhs);
    Customer& operator= (const Customer& rhs);
    ...
private:
    std::string name;
};

Customer::Customer(const Customer& rhs) : name(rhs.name) {
    logCall("Customer copy constructor");
}

Customer& Customer::operator=(const Customer& rhs) {
    logCall("Customer copy assignment operator");
    name = rhs.name;
    return *this;
}
```

到这里为止一切都很好，直到加入另外一个成员变量

```C++
class Date { ... };   // 日期
class Customer {
public:
    ...             // 同上
private:
    std::string name;
    Date lastTransaction;
};
```
这时候请注意了，上面的copying函数就变成了局部拷贝了，只拷贝了name而缺少了Transaction
而编译器也不会提醒你这个事实

一旦继承发生，就有潜藏危机

```C++
// 一个派生类
class PriorityCustomer: public Customer {
public:
    ...
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator= (const PriorityCustomer& rhs);
    ...
private:
    int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs) : priority(rhs.priority) {
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer& PriorityCustomer::operator= (const PriorityCustomer& rhs) {
    logCall("PriorityCustomer copy assignment operator");
    priority = rhs.priority;
    return *this;
}
```

PriorityCustomer 的 copying 函数复制了PriorityCustomer声明的成员变量
但是每个PriorityCustomer还内含了它继承的 Customer 成员变量的副本，而这些成员变量却未被复制
default构造函数将对 name 和 lastTransaction执行缺省的初始化操作

应该让derived class的copying函数调用相应的base class 函数：

```C++
// 增加一个操作：调用 base class 的copy构造函数
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs) : Customer(rhs), priority(rhs.priority) {
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer& PriorityCustomer::operator= (const PriorityCustomer& rhs) {
    logCall("PriorityCustomer copy assignment operator");
    // 对 base class 成分进行赋值操作
    Customer::operator=(rhs);
    priority = rhs.priority;
    return *this;
}
```

#### 总结
- Copying 函数应该确保复制“对象内的所有成员变量”以及“所有base class 成分”
- 不要尝试以某个copying函数实现另一个copying函数。应该将共同既能放在第三个函数中，并由两个coping函数共同调用。

## 资源管理

### 条款13： 以对象管理资源

对一个投资行为进行建模

``` C++
class Investment { ... };

// 返回指针，指向Investment继承体系内的动态分配对象，调用者负责删除它
Investment* createInvestment();

```

现在考虑使用f()来负责删除createInvestment返回的对象

```C++
void f() {
    Investment* pInv = createInvestment(); 
    ...
    delete pInv;
}
```

显然，最后一条语句未必可以执行到，因为在此之前就返回或者函数抛出异常，这样就产生了资源泄漏

为了确保createInvestment返回的资源总是可以被释放，我们需要将资源放进对象内
依赖于C++中的“析构函数自动调用机制”确保资源被释放

auto_ptr是一个特殊“类指针(pointer-like)对象”，即所谓的智能指针
其析构函数自动对其所指对象调用delete

```C++
void f() {
    std::auto_ptr<Investment> pInv(createInvestment());

    ...     // 经由auto_ptr的析构函数自动删除pInv
}
```

“以对象管理资源”的两个关键想法：
- 获得资源后立刻放进对象
- 管理对象运行析构函数确保资源被释放

C++有个底层条件：受auto_ptrs管理的资源必须绝对没有一个以上的auto_ptr同时指向它
否则，对象被删除一次以上，就会造成“未定义行为”

auto_ptr的替代方案是“引用计数型智能指针”

```C++
void f() {
    ...
    std::tr1::shared_ptr<Investment> pInv(createInvestment());
    ...
}
```

注意 auto_ptr 和 tr1::shared_prt两者都在析构函数内做delete而不是delete[]

#### 总结
- 为防止资源泄漏，请使用RAII对象，它们在构造函数中获得资源并在析构函数中释放资源
- 两个常被使用的RAII classes 分别是tr1::shared_ptr和auto_ptr。前者通常是较佳选择，因为其copy行为比较直观。若选择auto_ptr，复制动作会使它指向null
 
### 条款14： 在资源管理类中小心 copying 行为

有时候需要你自己建立一个资源管理类

假设建立一个class来管理锁机制

```C++
class Lock {
    public:
        explicit Lock(Mutex* pm) : mutexPtr(pm) {
            lock(mutexPtr);
        }
        ~Lock() {
            unlock(mutexPtr);
        }

    private:
        Mutex *mutexPtr;
}
```

使用时：
```C++
Mutex m;
...
｛
    Lock ml(&m);
    ...
｝
```

但是如果RAII对象被复制，则需要考虑两点
- 禁止复制
- 对底层资源使用“引用计数法”

使用tr1::shared_ptr时，计数器为0时，资源会被删除，但是我们可以对其指定自定义“删除器”

```C++
class Lock {
    public:
        explicit Lock(Mutex* pm) : mutexPtr(pm, unlock) {
            lock(mutexPtr.get());
        }
        ~Lock() {
            unlock(mutexPtr);
        }

    private:
        std::tr1::shared_ptr<Mutex> mutexPtr;
}
```

#### 总结
- 复制RAII对象必须一并复制它所管理的资源，所以资源的 copying 行为决定 RAII 对象的 copying 行为
- 普通而常见的 RAII class copying 行为是： 抑制 copying、施行引用计数法。不过其他行为也都可能被实现。

### 条款15： 在资源管理类中提供对原始资源的访问

由于许多APIs需要直接使用原始资源，所以RAII不得不提供一个访问原始资源的办法

```C++
std::tr1::shared_ptr<Investment> pInv(createInvestment());
int daysHeld(const Investment* pi);   // 返回投资的天数

int days = daysHeld(pIbv);   // 编译不通过
```

上面代码的最后一行编译不通过，因为daysHeld需要的是Investment*指针，但是却传了一个tr1::shared_ptr<Investment>对象

因此我们需要一个函数将RAII 对象转换成原始资源

#### 显式转换

```C++
int days = daysHeld(pInv.get());
```

#### 隐式转换

tr1::shared_prt 和 auto_prt 重载了指针取值操作符(operator-> 和 operator*)，允许隐式转换至底部的原始指针

```C++
class Investment {
    public:
        bool isTaxFree() const;
        ...

};

Investment* createInvestment();
std::tr1::shared_prt<Investment> pil(createInvestment());
bool taxablel = !(pil->isTaxFree());
...

std::auto_ptr<Investment> pi2(createInvestment());
bool taxable2 = !((*pi2).isTaxFree());
...
```

#### 总结：
- APIs往往要求访问原始资源(raw resources),所以每一个RAII class应该提供一个“取得其管理的资源”的办法
- 对原始资源的访问可能经由显式转换或者隐式转换。一般而言显式转换比较安全，但隐式转换对客户更加方便

### 条款16： 成对使用 new 和 delete 时要采取相同形式

```C++
std::string* stringArray = new std::string[100];
...
delete stringArray;
```
以上这个代码中，string数组中至少有99个对象不太可能被适当删除，因为它们的析构函数没有被调用

```C++
std::string* stringPtr1 = new std::string;
std::string* stringPrt2 = new std::string[100];
...
delete stringPtr1;
delete [ ] stringPtr2;
```

尽量不要对数组形式做typedefs动作

```C++
typedef std::string AddressLines[4];

// 此处返回一个string*，即“new string[4]”
std::string* pal = new AddressLines;

delete pal;         // 行为未定义
delete [ ] pal;     // 很好！

```

#### 总结：
- 如果你在new表达式中使用`[]`，必须在相应的delete表达式中也使用`[]`
- 如果你在new表达式中不使用`[]`，一定不要在相应的detele表达式中使用`[]`

### 条款17： 以独立语句将 newed 对象置入智能指针

假设一个函数用于处理程序优先权，另一个函数用来动态分配所得的Widget上进行某些带有优先权的处理

```C++
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);
```

当我们调用时

```C++

processWidget(new Widget, priority());

```

显然编译不通过

```C++
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
```

现在编译通过了，编译在调用processWidget会做三件事

1. 调用priority
2. 执行"new Widget"
3. 调用tr1::shared_ptr构造函数

操作2肯定会在操作3之前执行，但是操作1在什么时候执行这个问题弹性很大

假设按以下顺序执行，就有可能资源泄漏

1. 执行"new Widget"
2. 调用priority
3. 调用tr1::shared_ptr构造函数

万一调用priority时，抛出异常，“new Widget”的指针将会遗失


```C++
std::tr1::shared_ptr<Widget> pw(new Widget);

processWidget(pw, priority());
```

#### 总结：

以独立语句将 newed 对象存储于(置入)智能指针内。如果不这样做，一旦异常被抛出，有可能难以擦觉资源泄漏


## 设计与声明

### 条款18： 让接口容易被正确使用，不易被误用

理想上，如果客户企图使用某个接口却没有获得预期效果，这个代码不该通过编译；如果代码通过了编译，它的作为就是该客户所想要的。

可以使用导入简单的外覆类型(wrapper types)来区别年月日

```C++
struct Day {
explicit Day(int d) :val(d) { }

int val;
};

struct Month {
explicit Month(int m) :val(m) { }

int val;
};

struct Year {
explicit Year(int y) :val(y) { }

int val;
};

class Date {
    public:
        Date(const Month& m, const Day& d, const Year& y);
}

Date d(30, 3, 1995);  // 编译不通过
Date d(Day(30), Month(3), Year(1995));   // 编译不通过
Date d(Month(3), Day(30), Year(1995));   // 编译通过
```

对于值得有效性检验可以使用enums，但是不具备类型安全性
比较安全的做法是预先定义所有有效的Months

```C++
class Month {
public:
    static Month Jan() return Month(1);
    static Month Feb() return Month(2);
    ...
    static Month Dec() return Month(12);
    ...
private:
    explicit Month(int m);
    ...
};

Date d(Month::Mar(), Day(30), Year(1995));
```

#### 总结：
- 好的接口很容易被正确使用，不容易被误用。你应该在你所有接口中努力达成这些性质
- “促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容。
- “阻止误用”的办法包括建立新类型、限制类型上的操作，束缚对象值，以及消除客户的资源管理责任。
- tr1::shared_ptr支持定制型删除器(custom deleter)。这可以防范DLL问题，可被用来自动解除互斥锁等等。


### 条款19： 设计 class 犹如设计 type

如何设计高效的 classes?你需要思考一下一些问题：
- 新type的对象应该如何被创建和销毁
- 对象的初始化和对象的赋值该有什么样的差别
- 新type的对象如果被passed by value，意味着什么
- 什么是新type的“合法值”
- 你的新type需要配合某个继承图系吗？
- 你的新type需要什么样的转换？
- 什么样的操作符和函数对此新type而言是合理的
- 什么样的标准函数应该被驳回
- 谁该取用新type的成员
- 什么是新type的“未声明接口”
- 你的新type有多么一般化
- 你真的需要一个新type吗

#### 总结：

- class 的设计就是type的设计。

### 条款20： 宁以pass-by-reference-to-const 替换 pass-by-value



### 条款21： 必须返回对象时，别妄想返回其reference

### 条款22： 将成员变量声明为private

### 条款23： 宁以non-member、non-friend替换 member 函数

### 条款24： 若所有参数皆需类型转换，请为此采用 non-member 函数

### 条款25： 考虑写出一个不抛异常的 swap 函数

## 实现

### 条款26： 尽可能延后变量定义式的出现时间

### 条款27： 

### 条款28： 

### 条款29： 

### 条款30： 

### 条款31： 

## 继承与面向对象设计

### 条款32： 

### 条款33： 

### 条款34： 

### 条款35： 

### 条款36： 

### 条款37： 

### 条款38： 

### 条款39： 

### 条款40： 

## 模板与泛型编程


### 条款41： 

### 条款42： 

### 条款43： 

### 条款44： 

### 条款45： 

### 条款46： 

### 条款47： 

### 条款48： 

## 定制 new 和 delete

### 条款49： 

### 条款50： 

### 条款51： 

### 条款52： 

## 杂项讨论

### 条款53： 

### 条款54： 

### 条款55： 
