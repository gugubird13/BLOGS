---
title: 性能优化学习笔记2
date: 2024-03-31 20:31:58
tags: performance_engineering
categories: learning
description: 性能优化学习笔记
keywords: 性能优化、Bentley rules
top_img: https://s2.loli.net/2024/03/14/WQxHO96fkU5RJSe.jpg
cover: 
comments: false
---
# Class2——Bentley Rules for Optimizing Work

> 这篇文章主要是针对Program而不是体系结构，体系结构后面有专门的系列去记录
>
> 主要包括四个部分，数据结构、逻辑、循环、函数

## Data structures

------

### Packing and Encoding

**Packing的思想就是把多个数值存储在一个机器字里面，而Encoding的思想就是把一个数值用更少的bits去表示**

(主要思想就是更少的内存存取，memory access)

现在我们考虑对日期进行Encoding，这里我们假设仅仅对公元前4096到公元4096年之间进行编码，这里一共大约有 3M 的日期，如果我们用二进制进行存储，那么我们需要大约 $\lceil \log _2\left( 3\times 10^6 \right) \rceil \approx 22\ bits$ ，因此，这样我们仅仅需要一个word就可以存储，但是这样需要更多的工作去做解码，因为其不是一个显式的存储。需要的空间少了，工作加多了。

现在我们换种方式：

```c
typedef struct{
	int year: 13;  // 指示占用多少个比特
	int month: 4;
	int day: 5;
}day_t;
```

这样的pack的表示也仅仅占用22bits，现在我们能更快去访问每一个字段，这些字段仍然分别是以二进制的形式进行编码的。

(实际上C语言会对这个结构体进行padding操作，字对齐)

**而对于解包和解码操作呢？这些是不是浪费工作量的操作呢？**

其实不然，有的时候在工作涉及到移动数据或者操作数据，解包和解码操作可能会是优化工作

### Augmentation

数据结构扩容就是向数据结构添加额外信息，使得经常性的操作能够做更少的工作，比如说单链表的增加操作，我想把两个单链表连在一起，常见的操作就是某个单链表头指针走到最后，连上另一个单链表的头部即可，而数据结构的扩充操作，即可以增加一个**尾指针**



### Precomputation

>  **Precomputation**的思想就是提前做计算处理以避免在**关键任务时刻**去处理。

考虑二项式系数的程序，以下是二项式系数的表达式：
$$
\left( \begin{array}{c}
	n\\
	k\\
\end{array} \right) =\frac{n!}{k!\left( n-k \right) !}
$$
如果我想设计一个二项式计算函数，那么如果我仅仅这样做，计算量是非常大的(巨量的乘法计算)，并且还需要关注整数运算的溢出，因此，我们需要在初始化程序的时候，预先计算系数表，在我们运行的时候，需要二项式系数的时候，在这个表上查找即可。

这个表叫做帕斯卡三角：
$$
\left( \begin{matrix}{}
	1&		0&		0&		0&		0&		0&		0&		0&		0\\
	1&		1&		0&		0&		0&		0&		0&		0&		0\\
	1&		2&		1&		0&		0&		0&		0&		0&		0\\
	1&		3&		3&		1&		0&		0&		0&		0&		0\\
	1&		4&		6&		4&		1&		0&		0&		0&		0\\
	1&		5&		10&		10&		5&		1&		0&		0&		0\\
	1&		6&		15&		20&		15&		6&		1&		0&		0\\
	1&		7&		21&		35&		35&		21&		7&		1&		0\\
	1&		8&		28&		56&		70&		56&		28&		8&		1\\
\end{matrix} \right)
$$
对应的代码如下：

```c
int choose(int n, int k){
	if(n<k) return 0;
	if(n==0) return 1;
	if(k==0) return 1;
	return choose(n-1, k-1)+choose(n-1, k);
}
```

但是我们这样并没有实现一个真正的预计算，而是每次都要进行递归，这样的开销依然很大，那么我们应该如何做到预计算呢？

> Precomputing Pascal

```c
#define CHOOSE_SIZE 100
int choose[CHOOSE_SIZE][CHOOSE_SIZE];

void init_choose(){
    for(int n=0; n<CHOOSE_SIZE; ++n){
		choose[n][0]=1;
        choose[n][n]=1;
    }
    for(int n=1; n<CHOOSE_SIZE; ++n){
        choose[0][n] = 0;
        for(int k=1; k<n; ++k){
            choose[n][k] = choose[n-1][k-1] + choose[n-1][k];
            choose[k][n] = 0;
        }
    }
}
```

这样写，就可以在每次需要二项式的时候，做index索引操作即可；

那么，如果我们需要进行重复的运行程序，这样每次这段代码都会被运行，有没有方法使得这段代码仅仅被运行一次呢？

> compile-time initialization

我们在编译的时候就把值存储进去，即在编译的时候就把表存储进去即可。现在我的表格是10×10，那么如果我需要更大的表格，比如说100×100，我该如何做呢？我应该写一个程序，让这个程序帮我写程序，也就是元编程，像C++的模板、宏、JAVA的反射就是使用了元编程这个例子

### Caching

> Caching的思想即存储那些最近被访问的结果，这样程序不需要重复去计算这些结果(并非物理上面的存储，而是代码实现上的存储)

```c
double cached_A = 0.0;
double cached_B = 0.0;
double cached_h = 0.0;

// 计算三角形的斜边
inline double hypotenuse(double A, double B){
	if(A == cached_A && B == cached_B){
		return cached_h;
	}
	cached_A = A;
	cached_B = B;
	cached_h = sqrt(A*A + B*B);
	return cached_h; 
}
```

但是这样其实是不现实的，因为这里的代码中，cache的大小为1，我们可以制作一个大小为1000的cache，虽然耗费的查找时间长，但是仍然是值得的

### Sparsity

> Sparsity的思想就去避免存储和计算为0的值

考虑到矩阵与向量的乘法:
$$
\left( \begin{matrix}{}
	3&		0&		0&		0&		1&		0\\
	0&		4&		1&		0&		5&		9\\
	0&		0&		0&		2&		0&		6\\
	5&		0&		0&		3&		0&		0\\
	5&		0&		0&		0&		8&		0\\
	0&		0&		0&		9&		7&		0\\
\end{matrix} \right) \left( \begin{array}{l}
	1\\
	4\\
	2\\
	8\\
	5\\
	7\\
\end{array} \right)
$$
如果使用这样的矩阵直接相乘，那么需要 $n^2$ 次的乘法操作，而事实是，这个矩阵只有14个有效的元素，下面有一些优化方法：

1. 最简单的优化方法如下：在每次做运算的时候，检查是否为0，但是这样的检查仍然会造成开销

2. 使用CSR(Compressed Sparse Row)
   $$
   cols:\ \begin{matrix}{}
   	0&		4&		1&		2&		4&		5&		3&		5&		0&		3&		0&		4&		3&		4\\
   \end{matrix}
   $$

   $$
   vals:\ \begin{matrix}{}
   	3&		1&		4&		1&		5&		9&		2&		6&		5&		3&		5&		8&		9&		7\\
   \end{matrix}
   $$

   $$
   rows:\ \begin{matrix}{}
   	0&		2&		6&		8&		10&		11&		14\\
   \end{matrix}
   $$

   CSR中，cols 和 vals分别对应当前行的对应数据位置和数据值，而 rows存储的是各个行偏移位置；**这里一共是有三个array**

​	这样，对于CSR的方式进行存储，这里需要的存储空间就是： $$ n+nnz $$ ，其中 $ nnz $ 为矩阵中非0的数量， n即为矩阵的阶数

使用CSR进行矩阵乘法，核心思想就是对应的非零位置相乘就行：

```c
// CSR matrix-vector multiplication 

typedef struct{
	int n, nnz;
	int *rows;    // length n
	int *cols;    // length nnz
	double *vals; // length nnz
}sparse_matrix_t;

void spmv(sparse_matrix_t *A, double *x, double *y){
	for(int i=0; i<A->n; ++i){
		y[i] = 0;
        for(int k=A->rows[i]; k<A->rows[i+1]; ++k){
			int j= A->cols[k];
            y[i] += A->vals[k] * x[j];
        }
    }
}

// number of scalar multiplications = nnz, less than n^2
```

## Logic

------

### Constant Folding and Propagation

> 常量折叠和传播的原理是在编译过程中对常量表达式进行评估，并将评估结果代入其他表达式

我们来看这个例子：

```c
#inlcude <math.h>

// 太阳仪
void orrery(){
	const double radius=6371000.0;
	const double diameter=2*radius;
	const double circumference=M_PI*diameter;
	const double cross_area=M_PI*radius*radius;
	const double surface_area =circumference * diameter;
	const double volume=4*M_PI*radius*radius*radius /3;
	//...
}
```

这样写，在编译的时候，就可以进行评估(虽然现在编译器足够聪明，自己有的时候会进行自动评估，但是要养成这样的好习惯)

### Common-Subexpression Elimination

> 公共子表达式消除，我相信在编译原理这门课的时候都已经了解到

下面看例子：

``` c
a = b + c;
b = a - d;
c = b + c;
d = a - d;

// 可以改成下述形式

a = b + c;
b = a - d;
c = b + c;
d = b;
```

### Algebraic Identities

> 代数同构核心思想就是用计算成本更低的计算表达式去替换开销更昂贵的代数表达式

下面有一个这样的程序，我们想写一个程序去检测两个球的空间坐标，检测是否会重合(碰撞)：

```c
#include <stdbool.h>
#include <math.h>

typedef struct{
	double x;
	double y;
	double z;
	double r; // radius of ball
}ball_t;

double square(double x){
	return x * x;
}
	
bool collides(ball_t *b1, ball_t b2){
	double b = sqrt(square(b1->x - b2->x) 
					+ square(b1->y - b2->y)
					+ square(b1->z - b2->z));
	return d <= b1->r + b2->r;
} 

```

$$
\sqrt{u}\le v\ exactly\ when\ u\le v^2
$$

如果每次计算平方还不如两边都做平方运算，这样就可以减少运算量，**但是要注意溢出问题！！！**

### Short-Circuting

> 当我们执行一系列的测试的时候，不管是单元测试还是if语句，一旦我们知道答案是什么，后续其实就没必要进行下去了

```c
#include <stdbool.h>
// All elements of A are nonnegative
// check if the sum of A exceds the limit 
bool sum_exceds(int *A, int n, int limit){
    int sum = 0;
    for(int i=0; i<n; i++){
        sum += A[i];
    }
    return sum > limit;
}
```

很显然，优化的思想就是在计算的途中去检测，优化的代码如下：

```c
#include <stdbool.h>
// All elements of A are nonnegative
// check if the sum of A exceds the limit 
bool sum_exceds(int *A, int n, int limit){
    int sum = 0;
    for(int i=0; i<n; i++){
        sum += A[i];
        if(sum > limit)
			return ture;
    }
    return false;
}
```

这里需要讲一个知识点：

对于运算符： &&、|| 如果在表达式的左边就判断出结果了，那么他们就不会进行右边的运算(比如如果&&左侧为 0，那么整个式子都是0，没必要继续算右边了)

同时，如果你想通过表达式去图方便调用两个布尔函数，如果出现上述情况，右边的函数会不被调用，当心！

### Ordering Test

> Ordering tests: when ordering a series of tests, put the more often "successful" ones or computationally inexpensive ones first

这里，我们检查一个字符是否为空格，程序如下：

```c
#include <stdbool.h>
bool_is_whitespace(char c)
{
	if(c == '\r' || c == '\t' || c == ' ' || c == '\n'){
		return true;
	}
	return false
}
```

事实证明，空格和回车是要比制表符更加常见，因此可以这样改：

```c
#include <stdbool.h>
bool_is_whitespace(char c)
{
	if(c == ' ' || c == '\n' || c == '\t' || c == '\r'){
		return true;
	}
	return false
}
```

### Creating a Fast Path

让我们回到刚刚检测两个小球是否会碰撞问题，我们可以首先直接检查两个小球的bounding box是否相交，也就是把条件放得更加宽松一点：

```c
#include <stdbool.h>
#include <math.h>

typedef struct{
	double x;
	double y;
	double z;
	double r; // radius of ball
}ball_t;

double square(double x){
	return x * x;
}
	
bool collides(ball_t *b1, ball_t b2){
    if((abs(b1->x - b2->x) > (b1->r + b2->r)) ||
       (abs(b1->y - b2->y) > (b1->r + b2->r)) ||
       (abs(b1->z - b2->z) > (b1->r + b2->r)))
    
	double b = sqrt(square(b1->x - b2->x) 
					+ square(b1->y - b2->y)
					+ square(b1->z - b2->z));
	return d <= b1->r + b2->r;
} 

```

虽然上述程序很小，但是仍然有优化的地方，比如上述函数有很多被重复计算了，因此我们可以去先存储这些运算结果；同时这种思想特别在图形上有很好的应用，以提高图形程序的性能

### Combing Tests

> 用一个测试或者 switch去代替一系列的测试

这里有个全加器的实现例子：(Full Adder)

| a    | b    | c    | carry | sum  |
| ---- | ---- | ---- | ----- | ---- |
| 0    | 0    | 0    | 0     | 0    |
| 0    | 0    | 1    | 0     | 1    |
| 0    | 1    | 0    | 0     | 1    |
| 0    | 1    | 1    | 1     | 0    |
| 1    | 0    | 0    | 0     | 1    |
| 1    | 0    | 1    | 1     | 0    |
| 1    | 1    | 0    | 1     | 0    |
| 1    | 1    | 1    | 1     | 1    |

最麻烦的思想就是对每一对 $a,b,c$ 我都去做 $if$ 循环嵌套判断，从而给出答案，这很显然不符合人类的思维；正常的思维应该是：

这里一共有8种情况，也就是说，虽然 $a,b,c$ 都是一位的，但是他们组合起来就是一个三位的二进制数字，因此我们这样写：

```c
void full_add(int a,
			  int b,
			  int c,
			  int *sum,
			  int *carry){
	int test = ((a==1)<< 2) | ((b == 1)<< 1) | (c == 1);
	switch(test){
		case 0:
			*sum = 0;*carry = 0;
			break;
		case 1:
			*sum = 1;*carry = 0;
			break;
		case 2:
			*sum = 1;*carry = 0;
			break;
		case 3:
			*sum = 0;*carry = 1;
			break;
		case 4:
			*sum = 1;*carry = 0;
			break;
		Case 5:
			*sum = 0;*carry = 1;
			break;
		case 6:
			*sum = 0;*carry = 1;
			break;
		case 7:
			*Sum = 1;*carry = 1;
			break;
	}
}
```

同时，对于这个题目，有更好的方案去解决，就是预先计算这个结果，然后存在一张表里面，然后我在运行的时候只需要去查这张表就可以了。

## Loops

### Hoisting(吊起，我也不知道怎么翻译了)

> 其思想就是避免在循环中计算不变的量，这里不多赘述了，就是把不变的量提到循环外面

### Sentinels(哨兵)

> 放置在数据结构中的特殊虚拟值，用于简化边界条件的逻辑，特别是循环退出测试的处理

这里有个检验是否溢出的代码：

```c
#include <stdint.h>
#include <stdbool.h>

bool overflow(int64_t *A, size_t n){
// 所有的A中元素均为非负
    int64_t sum = 0;
    for(size_t i = 0; i < n; ++i){
        sum += A[i];
        if(sum < A[i]){
            return true;
        }
    }
    return false;
}
```

在上述代码中，每次迭代都做了两次的check，首先check是否应该退出循环体，后面进入循环体再判断是否溢出，以下是一个修改的程序：

```c
#include <stdint.h>
#include <stdbool.h>

// 我们假设A[n] 和 A[n+1] 都存在并且可以被赋值或者覆盖
bool overflow(int64_t *A, size_t n){
// 所有的A中元素均为非负
    A[n] = INT64_MAX;
    A[n+1] = 1; // 任何正数都可以
    size_t i = ;
    while(sum >= A[i]){
        sum += A[++i];
    }
    if(i<n) return true;
    return false;
}
```

这样，在每次迭代的时候，我只需要做一次check就可以了

### Loop unrolling and Loop fusion

> 循环展开和循环融合，循环展开的思想就是将一个循环的多个连续迭代合并为一个单一迭代 ，而循环融合的思想就是把同一个循环索引上面的循环合并成一个

循环展开分为两种：

1. 全循环展开：

   ```c
   // 求和循环展开，避免check////////////
   int sum = 0;
   for(int i=0;i<10;i++){
       sum += A[i];
   }
   
   //////////////////////////////////
   int sum=0;
   sum += A[0]; sum += A[1]; sum += A[2]; sum += A[3]; sum += A[4]; 
   sum += A[5]; sum += A[6]; sum += A[7]; sum += A[8]; sum += A[9]; 
   ```

   这个不是很常见，因为边界一般在编译的时候不确定并且循环次数没这么少

2. 部分循环展开：

   ```c
   int sum = 0;
   for(int i = 0; i< n; ++i){
       sum += A[i];
   }
   
   int sum = 0;
   int j = 0;
   for(j=0;j<n-3;j+=4){
       sum += A[j];
       sum += A[j+1];
       sum += A[j+2];
       sum += A[j+3];
   }
   
   // 不能被4整除的情况
   for(int i=j; i<n; ++i){
       sum += A[i];
   }
   ```

   这样做有两个好处，一个是减少了循环check 的次数，一个是单个循环体更大，这使得编译器能够优化代码。(但是如果展开太多，就会导致指令缓存会miss 的情况)

循环融合：

```c
for(int i=0;i<n;++i){
    C[i]=(A[i] <= B[i]) ? A[i] : B[i];
}
for(int i=0;i<n;++i){
    D[i]=(A[i] <= B[i]) ? B[i] : A[i];
}

//循环融合
for(int i=0;i<n;++i){
    C[i]=(A[i] <= B[i]) ? A[i] : B[i];
    D[i]=(A[i] <= B[i]) ? B[i] : A[i];
}
```

上述的融合不仅仅减少判断次数，同时也提供了更好的缓存局部性，因为当你计算C[i]的时候，其实A[i],B[i]就放在了缓存里面，这样计算D[i]的时候就可以减少开销

## Functions

### inline

> 第一次听到内联函数的时候是在学C++的时候，内联的思想就是函数体本身代替对函数的调用，从而避免函数调用的开销

编译器会试图在每个函数调用的地方将函数体直接插入，而不是通过常规的函数调用机制（如设置跳转地址，保存和恢复寄存器等）来调用函数。这样可以减少函数调用的开销，提高程序的执行效率。