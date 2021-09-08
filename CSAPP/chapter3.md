#### 程序编码

```shell
gcc -Og -o p p1.c p2.c
```

- -Og告诉编译器使用生成符合原始C代码整体结构的机器代码的优化等级
- 这部分指令是将p1.c, p2.c编译，然后链接生成可执行程序 p

#### 机器级代码

- 程序计数器PC，在x86-64用 %rip表示，给出将要执行的下一条指令在内存中的地址
- 整数寄存器文件包含16个命名的位置，分别存储64位的值。有的寄存器用来**记录某些重要的程序状态**，而其他的寄存器用来**保存临时数据**，例如过程的参数和局部变量，以及函数的返回值
- 条件码寄存器保存最近执行的算术或逻辑指令的状态信息。它们用来实现控制或数据流中的条件变化
- 一组向量寄存器可以存放一个或多个整数或浮点数值



C语言提供了一种模型，可在内存中声明和分配各种数据类型的对象。**但是机器代码只是简单地将内存看成一个很大的，按字节寻址的数组**。



#### 展示程序的字节表示

```shell
在gdb下：
(gdb) x/14xb multistore 
```

会从函数multistore所处地址开始的14个16进制格式表示

#### 查看机器代码文件的内容

使用反汇编器程序

```shell
objdump -d xxx.o
```

一些反汇编表示的特性：

- x86-64的指令长度从1到15个字节不等，**常用的指令以及操作数较少的指令所需的字节数少**，反之不常用的指令所需的字节数多
- 设计指令格式的方式是，从某个给定位置开始，可以将字节**唯一地解码**成机器指令
- 反汇编器只是**基于机器代码文件中的字节序列**来确定汇编代码。它不需要访问该程序的源代码或汇编代码
- 反汇编器使用的指令命名规则与GCC生成的汇编代码使用的有些区别

### 数据格式

| C 声明 | Intel数据类型 | 汇编代码后缀 | 大小（字节） |
| ------ | ------------- | ------------ | ------------ |
| Char   | 字节          | b            | 1            |
| short  | 字            | w            | 2            |
| int    | 双字          | l            | 4            |
| long   | 四字          | q            | 8            |
| Char*  | 四字          | q            | 8            |
| float  | 单精度        | s            | 4            |
| Double | 双精度        | l            | 8            |

虽然汇编代码里，int和double用的后缀一样，但是**因为浮点数使用的是一组完全不同的指令和寄存器**，因此不会造成歧义

### 访问信息

一个x86-64的中央处理单元，包含一组**16个存储64位值的通用目的寄存器**。

指令可以对这16个寄存器的低位字节中存放的**不同大小**的数据进行操作:

1. 字节级操作可以访问最低的字节
2. 16位操作可以访问最低的2个字节
3. 32位操作可以访问最低的4个字节



当指令以寄存器作为目标时，对于生成小于8字节结果的指令，有两条规则：

1. 生成1/2字节的指令，会保持剩下的字节不变
2. 生成4字节数字的指令会把高位4个字节置为0

#### 操作数指示符

各种不同的操作数的可能性被分为三种类型：

1. 立即数，用来表示常数值，书写方式是 $后面跟C表示法整数，如 $0x1F
2. 寄存器，表示某个寄存器的内容，低位1/2/4/8字节中的一个作为操作数
3. 内存引用，根据计算出来的地址，访问某个内存位置

#### 数据传送指令

数据传送指令主要是MOV类。

MOV S D（其中S代表source，D代表destination。

```shell
movl $0x4050, %eax
```

根据普通传送，源操作数是否做0扩展，源操作数是否做符号扩展又分为：

1. MOV
2. MOVZ
3. MOVS

而根据传送的数据大小，又可以分为（以MOV为例子）

1. movb（byte）
2. movw（word）
3. movl （long也就是双字）
4. movq
5. movabsq

#### 压入和弹出栈数据

栈顶元素是栈中元素地址中最低的

- pushq 将四字数据压入栈，等价于 R[&rsp]<-R[%rsp]-8, M[R[%rsp]] <- S
- popq 将四字弹出栈，等价于 D<-M[R[%rsp]] ，R[&rsp]<-R[%rsp]+8

### 算术和逻辑操作

#### 加载有效地址

加载有效地址(load effective address)指令，leaq不是将数据写入，而是将**有效地址写入到目的操作数**

> leaq 8(%rdi), %rax ==> R[%rax] = 8 + R[%rdi]
> movq 8(%rdi), %rax ==> R[%rax] = M[8 + R[%rdi]]
> 如果原来R[%rdi] = 0x100，M[0x108] = 2，那么leaq指令执行之后，寄存器%rax里面的值是0x108，而如果执行的是movq指令，那么寄存器%rax里面的值是2，即内存中地址为0x108的里面的值。
>
> 
>
> 作者：王商陆
> 链接：https://www.zhihu.com/question/40720890/answer/218717046
> 来源：知乎
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

## 控制

### 条件码

1. CF 进位标志，最近的操作使最高位产生了进位
2. ZF 零标志。最近的操作得出的过为0
3. SF 符号标志。最近的操作得到结果为负数
4. OF 溢出标志。最近的操作导致一个补码溢出

前面提到的一些指令（除了leaq），都会设置条件码

还有两类指令CMP（行为与SUB指令一致），TEST（行为与AND一致），只设置条件码，不改变寄存器状态

### 访问条件码

条件码通常不会直接读取，使用方法有三种：

1. 可以根据条件码的某种组合，将一个字节设置为0或1
2. 可以条件跳转到程序的某个其他的部分
3. 可以有条件地传送数据

参考书P137

### 跳转指令

常用的jmp指令是无条件跳转，而他又分为直接跳转和间接跳转：

1. 直接跳转，`jmp .L1` 直接跳转到L1的地方
2. 间接跳转，加多个*号，而其中又区分为两种写法：`jmp *%rax`指令用%rax的值作为跳转目标，`jmp *(%rax)`以rax中的值作为读地址，从内存中读出跳转目标。

除了jmp，我们还有其他有条件跳转指令

### 跳转指令的编码

并不是很懂这部分，在书P139

分为两种：

1. PC相对编码，它们会将目标指令的地址与紧跟在跳转指令后面那条指令的地址，之间的差作为编码
2. 给出绝对地址，直接指定目标



### 用条件控制来实现条件分支

GCC产生的汇编代码，会利用goto来实现分支跳转

### 用条件传送来实现条件分支

使用数据的条件转移：这种方法计算一个条件操作的两种结果，根据条件是否满足，从中选取一个，我们可以用**一条简单的条件传输指令**来实现它

原始代码：

```cpp
long absdiff(long x, long y){
  long result; 
  if(x<y)
    result = y - x; 
  else
    result = x - y; 
  return result;
}
```

使用条件赋值的实现

```cpp
long cmovdiff(long x, long y){
  long rval = y - x; 
  long eval = x - y; 
  long ntest = x >=y; 
  if(ntest) rval = eval; 
  return rval;
}
```

条件数据传送能加速的原因：

现代处理器通过流水线来获得高性能，流水线中，一条指令的处理要经过一系列阶段，每个阶段执行操作的一小部分（取指令，确定指令类型，从内存读数据，执行运算等）这种方法通过**重叠连续指令的步骤**来获得高性能。

如取一条指令的同时，执行它前面一条指令的条件运算

要做到这一点，要求能够**事先确定要执行的指令序列**，才能保持流水线中充满待执行的指令。

机器遇到条件跳转时，只有当分支条件求值完成，才能决定分支往哪儿走，处理器会采用**分支预测逻辑**来猜测每条指令是否会执行。当**错误预测一个跳转**，则要求处理器**丢掉它为该跳转指令后所有指令已做的工作**

条件传送指令就是CMOV系列（可参考书P147）

更抽象一点，我们可以看条件控制：

```cpp
v = test-expr ? then-expr : else-expr; 
// 汇编
if(!test-expr)
  goto false;
v = then-expr; 
goto done; 
false: 
	v = else-expr; 
done:
```

换成条件数据传输，则是

```cpp
v = then-expr; 
ve = else-expr; 
t = test-expr; 
if(!t) v = ve; 
return v;
```

使用条件传送**不总是提高代码效率**，如果then-expr或者else-expr的求值需要大量的计算，当对应条件不满足时，这些工作就白费了。

### 循环

循环的实现在汇编也是改用goto来做到的

#### do while循环

```cpp
long result = 1 
do{
  result *= n; 
  n = n-1;
} while(n>1); 

// 汇编
long result = 1; 
loop: 
	result *= n; 
  n = n - 1; 
if (n>1)
  	goto loop;
return result; 
```

#### while循环

while循环的翻译用两种。

1. Jump to middle:

```cpp
	goto test;
loop: 
	body-statement; 
test: 
	t = test-expr; 
	if(t)
    goto loop; 
```

2. Guarded-do

首先用条件分支，如果初始条件不成立就跳过循环，把代码变换为do-while循环

```cpp
t = test-expr; 
if(!t)
  goto done; 
loop: 
	body-statement
  t = test-expr;
	if(t)
    goto loop;
done:
```

#### for循环

是while循环的两种翻译之一，不再赘述

#### switch语句

是一个多重分支语句，通过使用跳转表这个数据结构使得实现更加高效。

当开关数量比较多，并且值的范围跨度比较小时，就会使用跳转表。

> GCC作者创造一个运算符&&，指向代码位置的指针

```cpp
switch(n){
  case 100: 
  case 102: 
  case 103: 
  case 104: 
  case 106: 
  default
}

// 跳转表
static void *jt[7] = {
  &&loc_A, &&loc_def, &&loc_B, &&loc_C, &&loc_D,  &&loc_def, &&loc_D
}
```

由于case从100-106，加上额外的default，我们需要声明一个含7个元素的数组

### 运行时栈

大多数编程语言的过程调用机制在于使用了栈数据结构的后进先出的内存管理机制

当过程需要的存储空间超出寄存器能够存放的大小时，就会在栈上分配空间，这个部分称为过程的栈帧

### 转移控制

将控制从函数P转移到函数Q，只需简单地把程序计数器PC设置为Q的代码的起始位置。

当稍后从Q返回的时候，处理器必须记录好它需要继续P的执行的代码位置

这个信息用指令 `call Q`来记录，它会：

- 将地址A压入栈中，将PC设置为Q的起始记录
- 压入的地址A被称为返回地址

对应的指令ret会从栈中弹出地址A，并把PC设置为A

### 数据传送

X86-64中，大部分过程间的数据传送是通过寄存器实现的

我们可以通过寄存器最多传递**6个整型参数**

1. 寄存器的使用有特殊顺序
2. 寄存器使用的名字取决于要传递的数据类型大小

如果一个函数有大于6个整型参数，**超出6个的部分就要通过栈传递**

当有n个整型参数（n>6），需要把1-6复制到对应的寄存器，把参数7-n放到栈上，而参数7位于栈顶

### 栈上的局部存储

有些时候，局部数据必须存放在内存中：

- 寄存器不足够存放所有的本地数据
- 对一个局部变量使用地址运算符&，因此必须能够为它产生一个地址
- 某些局部变量是数组或结构体，得通过数组或结构引用被访问到

以书P170页为例子

我们写了一个swap函数，其接受两个long类型参数

首先栈指针需要减掉16，分别给两个long类型分配栈帧

完成运算后再加回16

### 寄存器中的局部存储空间

寄存器组是唯一被所有过程共享的资源，虽然给定时刻只有一个过程是活动的，我们仍然需要确保党一个过程调用另一个过程时，被调用者不会覆盖调用者稍后会使用的寄存器值

- 寄存器rbx，rbp，r12-r15被划分为被调用者保存寄存器

当过程P调用过程Q时，Q必须保存这些寄存器的值，以保证它们的值在Q返回到P时与Q被调用时是一样的

Q要么是不改变寄存器的值，要么就是把**原始值压入栈中，改变寄存器的值**

### 递归过程

递归调用一个函数本身与调用其他函数是一样的

每次函数调用都有它自己私有的状态信息存储空间。

## 数组分配和访问

C语言中的数组是一种将标量数据聚集成更大数据类型的方式

它可以产生指向数组中元素的指针，并对这些指针进行运算

### 基本原则

对于一个数组，会一次性分配指定的内存空间

X86-64的内存引用指令可以用来简化数组访问

```cpp
movl(%rdx, %rcx, 4) %eax
```

假设E是一个**int数组**，我们想计算E[i]

那么会将E地址放到rdx，i存放在rcx，最后加了个4

即把E[4i]的元素存放到eax（因为一个int是占4个字节）

### 嵌套的数组

```cpp
int A[5][3]
```

实际排布还是以一维形式，索引计算方法如下：

```cpp
假设有一个数组 D[R][C]
那么&D[i][j] = x0 + L*(i* C + j)
```

### 定长数组

以一个矩阵乘的例子，展示下GCC的优化

```cpp
// 原代码
int matrix_mul(matrix A, matrix B, long i, long k){
  long j;
  int result = 0; 
  for(j = 0; j < N; j++){
    result += A[i][j] * B[j][k]
  } 
}
// 优化后
int *Aptr = &A[i][0]; 
int *Bptr = &B[0][k];
int *Bend = &B[N][k];
int result = 0; 
do{
  result += *Aptr * *Bptr; 
  Aptr ++; 
  Bptr += N; 
} while (Bptr != Bend)
```

1. gcc去掉整数索引，并把所有的数组引用转换成指针间接引用

### 数据对齐

计算机系统对基本数据类型的合法地址做出一些限制

地址必须是某个值K（2/4/8）的倍数。

假设处理器总是从内存取8个字节，在double数据地址对齐成8的倍数情况下，读一次double只需要一次内存访问。如果没对齐，则需要执行两次内存访问。

对于结构体代码，编译器可能**插入间隙**来保证对齐

以

```cpp
struct S1 {
  int i; 
  char c; 
  int j; 
}
```

按照我们的思想，可能是这样分配的

```
0 - 4 - 5 - 9
  i   c   j
```

但这样不满足4字节对齐要求，所以编译器会给char多插3字节空的，以对齐

```
0 - 4 - 8 - 12
  i   c   j
```
