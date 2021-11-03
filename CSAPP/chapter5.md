### 优化编译器的能力和局限性

我们仅考虑GCC -o1产生的代码

编译器必须很小心地对程序只使用**安全的优化**

考虑下面两段代码

```cpp
void func1(){
  *xp += *yp; 
  *xp += *yp;
}
```

```cpp
void func1(){
  *xp += 2* *yp; 
}
```

乍一看两者作用是一样的，但存在这么一种情况，当yp == xp，那么func1是变为4倍，func2是变为3倍



这种两个指针可能指向同一个内存位置的情况称为**内存别名使用**。在只执行安全的优化中，编译器必须假设不同的指针可能会指向内存中同一个位置。这就是一个妨碍优化的因素。



第二个妨碍优化的因素是**函数调用**，考虑下面代码：

```cpp
long f(); 

long func1(){
  return f() + f() + f() + f(); 
}

long func2(){
  return 4*f(); 
}
```

如果f函数存在改变全局程序状态的副作用，如

```cpp
long counter = 0; 
long f(){
  return counter++; 
}
```

那么这两个函数就不等价，func1变成 0 + 1 + 2 + 3，func2还是4 * 0。

#### tips：用内联函数替换优化函数调用

首先我们用内联函数去替换func1

```cpp
long func1inline(){
  long t = counter++; 
  t += counter++; 
  t += counter++; 
  t += counter++; 
  return t; 
}
```

**这样的转换减少了函数调用的开销，也允许对展开的代码做进一步优化**

```cpp
long func1Opt(){
  long t = 4*counter + 6; 
  counter += 4; 
  return t;
}
```

经过内联替换后，对这个函数调用进行追踪或设置断点的尝试都会失败

### 表示程序性能

我们引入 **每元素的周期数 CPE** 来作为一种表示程序性能，并指导我们改进代码的方法

### 程序示例

这里只截取一部分

```cpp
void combine1(vec_ptr v, data_t *dest){
  long i; 
  
  *dest = IDENT; 
  for(i = 0; i < vec_length(v); i++){
    data_t val; 
    get_vec_element(v, i, &val); 
    *dest = *dest OP val; 
  }
}
```

一般开启 -O1 选项进行一些基本的优化，就可以显著地提高这段程序的性能

### 消除循环的低效率

可以看到我们在for循环内，每次都调用了`vec_length`这个方法来得到向量的长度。

我们可以知道这个长度其实是固定，只需要在外部求一次即可

```cpp
void combine2(vec_ptr v, data_t *dest){
  long i; 
  
  *dest = IDENT; 
  long length = vec_length(v);
  
  for(i = 0; i < length; i++){
    data_t val; 
    get_vec_element(v, i, &val); 
    *dest = *dest OP val; 
  }
}
```

这种优化称为代码移动，这类优化包括识别要执行多次但是计算结果不会改变的计算。



然而跟前面的例子一样，编译器不能可靠地发现这个函数调用是否会有其他**副作用**，这样两个函数是不等价的，因此需要程序员手动地做代码移动。



### 减少过程调用

这里我们试图消除combine2函数中for循环里调用的`get_vec_element`方法，于是有下面的版本

```cpp
data_t * get_vec_start(vec_ptr v){
  return v->data; 
}

void combine3(vec_ptr v, data_t *dest){
  long i; 
  long length = vec_length(v);
  data_t *data = get_vec_start(v); 
  
  *dest = IDENT; 
  for(i = 0; i < length; i++){
    *dest = *dest OP data[i]; 
  }
}
```

我们直接替换为数据访问，但是并没有提升，显然是**内循环的其他操作形成了瓶颈**

### 消除不必要的内存引用

combine3的代码在每次循环，都将值**累积到指针dest指定的位置**，可以看到每次迭代的时候，都需要将累积变量dest从内存读出，再写入到内存。



消除这种内存读写就是提供一个临时变量，然后最后再写入到指针内

```cpp
void combine4(vec_ptr v, data_t *dest){
  long i; 
  long length = vec_length(v);
  data_t *data = get_vec_start(v); 
  data_t acc = IDENT; 
  
  
  for(i = 0; i < length; i++){
    acc = acc OP data[i]; 
  }
  *dest = acc; 
}
```

## 理解现代处理器

在实际的处理器中，cpu是**同时对多条指令求值的**，这个现象称为指令级并行。CPU采用一些机制让指令并行地执行，又呈现出一种简单的顺序执行指令的表象

两种下界描述了程序的最大性能：

- 当一系列操作必须按照严格顺序执行时，就会遇到延迟界限，在下一条指令开始之前，这条指令必须结束
- 吞吐量界限刻画了处理器功能单元的原始计算能力

   ## 循环展开

循环展开是一种程序变换，通过增加每次迭代计算的元素的数量，减少循环的迭代次数

它从以下两个方面来改进程序的性能：

- 减少了不直接有助于程序结果的操作的数量，如循环索引计算和条件分支
- 提供一些方法进一步变化代码，减少整个计算中关键路径的操作数量

如果我们一次操作两个数目，还需要确保元素总个数不被2整除，对应的伪代码如下：

```cpp
length = vec_length; 
limit = length - 1; 

for(i=0; i < limit; i+=2){
  acc = (acc OP data[i]) OP data[i+1]
}

// 完成剩余的元素
for(;, i < length; i++){
  acc = acc OP data[i]; 
}
```

## 提高并行性

### 多个累积变量

对于加法，乘法这种可结合和可交换的合并运算来说，我们可以分割成多个部分分别计算部分结果，最后合并部分结果来提升性能。

### 重新结合变换

```cpp
acc = (acc OP data[i]) OP data[i+1]
// 变成
acc = acc OP (data[i] OP data[i+1])
```

## 一些限制因素

### 寄存器溢出

循环并行性的好处受汇编代码描述计算的能力限制。

如果我们的并行度超过了可用的寄存器数量（x86-64有16个寄存器），那么编译器会溢出，将某些临时值放到内存

### 分支预测和预测错误处罚

保证分支预测处罚不会阻碍程序的效率有以下通用原则：

1. 不要过分关心可预测的分支
2. 书写适合使用条件传送实现的代码

当然不是所有条件行为都可以使用条件数据传送来实现

## 理解内存性能

存储单元包含一个存储缓冲区，它包含已经被发射到存储单元而又还没有完成的存储操作的地址和数据，这里的完成包括更新数据高速缓存

