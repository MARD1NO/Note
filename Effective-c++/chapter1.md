### 条款1：视C++为一个语言联邦

我们可以把C++看成以下几种语言的联邦：

- C， C++依旧是以C语言为基础，
- Object-Oriented C++，加入了面向对象特性
- Template C++，是C++泛型编程部分
- STL， 对容器，迭代器，算法，函数对象的规约有紧密配合与协调，如果使用STL需要遵守其设计的规定。

> C++高效编程守则视状况而变化，取决于你使用C++哪一部分

例如对于C内置类型，值传递通常高效点。

而对于Object-Oriented C++，由于构造函数和析构函数的存在，通常引用传递会更好



### 条款2：尽量以const，enum，inline来替换#define

#define不是语言的一部分

假设有一段代码

```c++
#define ASPECT_RATIO 1.653
```

记号`ASPECT_RATIO`也许没被编译器看见，也许处理源码的时候已经被预处理器移走了，当相关代码出现编译错误信息的时候，它提示的会是1.653，而不是这个记号，难以debug



我们讨论两种情况：

- 情况1：定义常量指针，需要把常量和指针都作为const，所以const一般要写两次

```cpp
const char* const authorName = "xxx"
```

更推荐用`std::string`

```cpp
const std::string authorName("xxx")
```

- 情况2：class专属常量，我们需要将其作用域限制在class，此外为确保常量至多只有一份，我们需要让他成为一个static成员

```cpp
class X{
private: 
    static const int NumTurns = 5; // 常量声明式
    int scores[NumTurns];
}
```

如果你取某个class专属常量的地址，或编译器坚持看到一个定义式，需要提供定义式如下：

```cpp
const int X::NumTurns; // NumTurns定义
```

若编译器不支持上述语法，可以将初值放在定义式内。



有一个例外情况是，class编译期间需要一个常量值(比如上面的array声明式)，若编译器不允许这种做法，我们可以改用enum

```cpp 
class X{
private: 
    enum {NumTurns = 5};
    int scores[NumTurns];
}
```

并且取一个enum地址不合法



下面看一个通过宏实现函数的错误例子

```cpp
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

考虑以下例子：

```cpp 
int a = 5, b = 0; 
CALL_WITH_MAX(++a, b); // 首先在f(a)那里被+1，由于a大，所以(a)这里又被+1，因此就是+2
CALL_WITH_MAX(++a, b+10); // 只有在f(a)被+1
```

我们可以通过template inline函数来得到宏的效率+函数的安全性

```cpp 
template<typename T>
inline void callWithMax(const T& a, const T& b){
    f(a > b ? a: b);
}
```

- 对于单纯常量，最好以const或enum替换#define
- 对于形似函数的宏，最好使用inline函数替换

### 条款3：尽可能使用const

const修饰原则：**const默认作用其左边的东西，否则作用于其右边的东西** （https://www.zhihu.com/question/443195492/answer/1723886545）

```cpp
const char* p = "hello"; // const没左边的东西，所以作用在char，const char, non const pointer
char* const p = "hello"; // const作用在*，说明pointer是const，data不是
```



迭代器作用就是个T*指针，若声明迭代器为const，则等价于

```cpp
T* const 
```

表示该指针是const，若想让迭代器所指向的内容不可被改动，则使用`const iterator`。 



令函数返回一个常数值，通常可以避免大量错误

```cpp 
class Rational{...}; 
const Rational Operator*(const Rational& lhs, const Rational& rhs);
```

如果有人误写代码：

```cpp 
if(a*b=c) ..。
```

const返回值能预防赋值这个无意义的操作。



####  const成员函数

实现这类成员函数有两个理由：

1. 使class接口容易被理解，能够得知哪个函数可以改动对象内容而哪个函数不行
2. 使操作const对象成为可能，改善程序效率



const分为两派：

- bitwise const：成员函数只有在不更改对象之任何成员变量才算const
- logical const：一个const成员函数可以修改对象内的某些bits

> 函数声明后面跟一个const，表面这个函数不会修改成员变量。

使用logical const可能会导致编译器报错，我们可以利用`mutable`关键字，来修饰某些变量

#### 在const和non-const成员函数中避免重复

假设我们有这样一段代码：

```cpp
class TextBlock{
public: 
    const char& operator[](...) const{
        ... // 边界检查
        ... // 日志数据访问
        ... // 检验数据完整性
    }
    char& operator[](...) {
        ... // 边界检查
        ... // 日志数据访问
        ... // 检验数据完整性
    }
}
```

虽然我们可以把检查逻辑封装到一个函数内去调用，但还是重复了一些代码，我们需要实现其中一个调用另一个，促使我们将**常量性转除**



就一般二眼，cast并不是一个好想法。本例中，`const operator[]`完全做了non-const版本的一切，如果将返回值的const移除是安全的，那我们也可以安全地代码复用

```cpp
class TextBlock{
public: 
    const char& operator[](...) const {
        ...
    }
    char& operator[](...){
        return 
            const_cast<char&>( // 将返回值的const移除
        		static_cast<const TextBlock&>(*this)[position]; // 给*this加上const，调用const op[]
        )
    }
}
```

这份代码有两个转型动作：

1. non-const operator[] 会调用const版本，
2. 使用static-cast移除const

注意我们调用的方向是:non-const版本调用const版本，反过来则会有一定风险

- 将某些东西声明为const可帮助编译器侦测错误用法
- 编译器强制实施bitwise constness，但我们编写程序应使用概念上的常量性
- 当const和non-const代码有实质等价的实现时，令non-const版本调用const版本可避免代码重复

### 条款4：确定对象被使用前已被初始化

简单来说，就是给对象及其成员进行初始化

C的array不保证其内容初始化，而c++的vector有此保证

所以最佳处理方法是：**使用对象前将其初始化**



注意不要混淆**赋值**和**初始化**

```cpp
class ABEntry{
public: 
    ABEntry(const std::string& name, const std::string& address);
private: 
    std::string theName; 
    std::string theAddress; 
    int numTimesConsulted; 
}; 

ABEntry::ABEntry(const std::string& name, const std::string& address){
    theName = name; 
    theAddress = address; // 这些都是赋值，而不是初始化
}

```

C++规定，**对象的成员变量的初始化动作发生在进入构造函数本体之前**

构造函数的一个较好的写法是，用**成员初始化列表**来完成

```cpp
ABEntry::ABEntry(const std::string& name, const std::string& address): theName(name), theAddress(address), numTimesConsulted(0){
    
}
```

上面这些就属于初始化了，通常**效率较高**



而我们本例的第一个版本，**是先调用default构造函数设置初值，再立刻对它们赋予新值**

而以成员初始化列表操作的版本，是直接copy赋值（对于numTimesConsulted变量，其初始化和赋值成本相同）

尽管成本相同，但为了一致性最好也通过初始化列表来完成：

```cpp
ABEntry::ABEntry(): theName(), theAddress(), numTimesConsulted(0){}
```

我们规定在初始化列表列出所有成员变量，以免遗漏某些成员变量，引发未定义行为。

变量初始化顺序是按照初始化列表来的，为了避免错误，**变量初始化顺序与变量声明顺序保持一致**











