### 条款05：了解C++默默编写并调用了哪些函数

编译器会自动帮我们声明：

- 一个copy构造函数
- 一个copy assignment
- 一个析构函数

这些函数都是public且inline的

注意的是，编译器产出的析构函数是个non-virtual.

而copy构造函数和copy assignent操作符，只是单纯地复制每一个non-static成员

如果赋值操作中涉及处理const，引用的成员。那么默认的赋值操作符就不满足，c++会拒绝编译

### 条款06 若不想使用编译器自动生成的函数，应该明确拒绝

一般来说，我们不想让class支持某一个功能，那么就不声明对应的函数即可。但是编译器针对copy构造和copy assignment会生成默认的函数，一个简单的解决方法是，将其放到private内部。

```cpp
class HomeForSale{
public:
  ...
private: 
  HomeForSale(const HomeForSale&);
  HomeForSale& operator=(const HomeForSale&);
}
```

另外我们可以构造一个Uncopyable类，让别的类从这里继承

```cpp
Class Uncopyable{
protected: 
  Uncopyable(){}
  ~Uncopyable(){}
private: 
  Uncopyable(const Uncopyable&);
  Uncopyable& operator=(const Uncopyable&);
}
```

### 条款07 为多态基类声明virtual析构函数

假设我们有一个基类以及对应的派生类

```cpp
class TimeKeeper{
public:
  TimeKeeper();
  ~TimeKeeper();
}; // base class
class AtomicClock: public TimeKeeper{...}; 
class WaterClock: public TimeKeeper{...};
...
```

我们设计一个工厂函数，来返回不同的对象

```cpp
TimeKeeper* getTimeKeeper(); // 返回一个指针，指向一个TimeKeeper派生类的动态分配对象
TimeKeeper* ptk = getTimeKeeper(); 
delete ptk; // may wrong!
```

析构的时候问题发生，原因是**getTimeKeeper返回的是一个派生类对象，而最后析构却由一个base class来完成**

正确做法是：**给base class提供一个virtual析构函数**

```cpp
~TimeKeeper();
```

当class不是作为一个base class，则不建议使用virtual析构函数

> 实现virtual函数，要求对象携带某些信息，用来在运行期决定哪一个virtual函数被调用。这份信息是由一个virtual table pointer指针指出，vptr是一个由函数指针构成的数组，成为virtual table虚表。

如果class含virtual函数，则会携带vptr信息，让对象体积增加

tips：**当class内含至少一个virtual函数，才为它声明virtual析构函数**

另外需要注意的是，一些base class的设计并不是用于多态，这类也不需要virtual析构函数（比如上一节用的Uncopyable类）

- 带多态性质的base class应该声明一个virtual析构函数
- 如果不是作为base class或不具备多态，不该声明virtual析构函数

### 条款08 别让异常逃离析构函数

c++不喜欢析构函数吐出异常：这样会让其没有完整的销毁对象，发生资源泄漏。

一个简单方法是try catch捕捉

```cpp
try{db.close();} 
catch(...){
  xxx; 
  std::abort();
}
```

另外我们可以设计一个更好的策略，单独提供一个close函数，让客户有机会处理。另外在析构函数中，我们仍保持try catch逻辑，上双保险

```cpp
void close(){
  db.close(); 
  closed = true; 
}
~DBConn(){
  if(!closed){
    try{
      ...
    }
    catch(...){
      ...
    }
  }
}
```

- 如果析构函数可能抛出异常，则需要保证捕捉任何异常
- 如果调用者需要对抛出的异常做自定义处理，则需要提供一个普通函数接口，给调用者，而不是完全封装在析构函数中

### 条款09 绝不在构造和析构过程中调用virtual函数

```cpp
class Transaction{
public: 
  Transaction(){
    logTransaction(); 
    ...
  }; 
  virtual void logTransaction() const = 0;
};
class BuyTransaction: public Transaction{
public: 
  virtual void logTransaction() const; 
};
BuyTransaction b; // Wrong!
```

此时我们构在一个对象b，他首先调用的是其基类的构造函数，**问题出来，基类的构造函数中，调用的logTransaction是base class版本的，而不是BuyTransaction的**

正确的做法是改成non-virtual，并在派生类传递信息给base class

```cpp
class Transaction{
public: 
  explicit Transaction(const std::string& logInfo){
    logTransaction(logInfo); 
    ...
  };
  void logTransaction(const std::string& logInfo) const;
}; 
class BuyTransaction : public Transaction{
public: 
  BuyTransaction(params): Transaction(createLogString(params)){
    ...
  }
private: 
  static std::string createLogString(params); 
};
```

### 条款10 令operator= 返回一个reference to *this

这么做的目的是为了实现连锁赋值

```cpp
int x, y, z; 
x = y = z = 10;
```

做法就是

```cpp
Widget& operator=(const Widget& rhs){
  ...
  return *this;
}
```

### 条款11 在operator= 中处理自我赋值

自我赋值可能发生在很隐秘的地方

```cpp
a[i] = a[j]; // 如果i==j就是自我赋值
*px = *py; // 如果px py指向的是一个对象，也是自我赋值
```

假设我们有以下class

```cpp
class Bitmap{...}; 
class Widget{
  ...
private: 
  Bitmap* pb; 
}
```

处理自我赋值只需我们调整好逻辑：我们只需要注意在复制pb所指向的对象前，别删除pb即可

```cpp
Widget& Widget::operator=(const Widget& rhs){
  Bitmap* pOrig = pb; 
  pb = new Bitmap(rhs.pb); 
  delete pOrig; 
  return *this; 
}
```

### 条款12 复制对象时勿忘其每一个成分

拷贝我们有copy构造函数和copy assignment运算符，这两个我们都需要注意复制的时候，每一个成员都要复制到位

```cpp
class Customer{
  ...
}; 
class PriorityCustomer: public Customer{
public: 
  ...
private: 
  int priority; 
}; 
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs): Customer(rhs), priority(rhs.priority){
  ...
}
PriorityCustomer& PriorityCustomer::operator=(const PriorityCustomer& rhs){
  Customer::operator=(rhs); // 对base class赋值 
  priority = rhs.priority;
  ...
  return *this; // 条款11
}
```





