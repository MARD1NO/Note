### 条款13 以对象管理资源

```cpp
void f(){
  Investment* pInv = createInvestment(); 
  ...
  delete pInv; 
}
```

在复杂的情况下，...部分可能过早return，也可能抛出异常，导致delete没被正确执行，没有释放对象。

一个原则是：**把资源放进对象内，我们便可依赖析构函数来确保资源释放**

虽然书上用的是auto_ptr，但是搜了下别的资料，应该用shared_ptr

```cpp
void f(){
  std::shared_ptr<Investment>pInv(createInvestment()); 
}
```

两个关键的想法：

- **获得资源后立刻放进管理对象内**，这种观念也被称为RAII，资源取得时机便是初始化时机（Resource Acquisition Is Initialization）
- **管理对象运行析构函数确保资源被释放**

另外要注意的是，shared_ptr的析构都是delete，**因此用其管理动态array并不是个好想法（需要delete [])**，通常vector模板类都能满足我们动态array的需求

### 条款14 在资源管理类中小心copying行为

考虑我们有一个Lock类代码

```cpp
class Lock{
public: 
  explicit Lock(Mutex* pm): mutexPtr(pm){lock(mutexPtr); }
  ~Lock() {unlock(mutexPtr); }
private: 
  Mutex *mutexPtr; 
};

Lock ml1(...); 
Lock ml2(ml1); // what will happen?
```

对于一个RAII对象被复制时，我们有以下两个选择

- 禁止复制
- 对底层资源使用引用计数

针对第二点，我们仅需用shared_ptr管理即可，另外其析构函数应当是unlock而不是delete，所以我们要传入一个删除器件

```cpp
class Lock{
public: 
  explicit Lock(Mutex* pm): mutexPtr(pm, unlock){ // 在mutexPtr传入删除器，指定用unlock
    lock(mutexPtr.get()); 
  }
private: 
  std::shared_ptr<Mutex> mutexPtr; 
};
```

### 条款15 在资源管理类中提供对原始资源的访问

```cpp
std::shared_ptr<Investment> pInv(createInvestment());

int daysHeld(const Investment* pi); // 存在这个函数
int days = daysHeld(pInv); // 报错
```

这部分通不过编译的原因是daysHeld需要的是Investment指针，而pInv是一个shared_ptr。

解决方法也很简单，智能指针提供了.get()方法，返回智能指针内部的原始指针

```cpp
int days = daysHeld(pInv.get()); 
```

在资源管理类中，我们也应当提供get方法，来返回资源

### 条款16 成对使用new和delete时要采取相同形式

这个也很简单，即

```cpp
new xxx -> delete xxx
new xx[100] -> delete [] xxx
```

对于单一对象时，我们new的时候，内存分配是

```cpp
object
```

而对于动态数组，绝大多数编译器处理是记录数组大小，接着是一系列对象

```cpp
n object object object object
```

当使用delete[]，他会读取一定字节，得到这个数组有多少个对象，再一个个释放

另外动态数组的需求可以考虑使用vector模板类实现，这样更方便安全

### 条款17 以独立语句将newed对象置入智能指针

```cpp
processWidget(std::shared_ptr<Widget>(new Widget), priority()); 
```

当放到一句时候，执行顺序可能被打乱，可能有如下这个顺序：

1. 执行 new Widget
2. 调用 priority
3. 调用 shared_ptr构造函数

当priority执行抛出异常，那么先前的new Widget返回的指针会遗失。

解决方法很简单，拆开成独立语句，强迫编译器按顺序执行

```cpp
std::shared_ptr<Widget> pw(new Widget); 
processWidget(pw, priority()); 
```



