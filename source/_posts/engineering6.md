---
title: 性能优化学习笔记6
date: 2024-05-21 15:17:56
tags: performance_engineering
categories: learning
description: 性能优化学习笔记
keywords: 性能优化、多核编程
top_img: https://s2.loli.net/2024/03/14/WQxHO96fkU5RJSe.jpg
cover: 
comments: false
---
# 多核编程

> 这里的引言就是随着摩尔定律的终结，单核处理器的性能提升已经非常有限，而多核处理器的出现为我们提供了一种新的性能提升途径。多核处理器的出现，使得我们可以通过并行的方式来提高程序的性能，但是多核编程也带来了一些新的问题，比如线程之间的通信、数据共享、数据一致性等问题。本文将介绍多核编程的一些基本概念和技术，希望对大家有所帮助。

这门课学到的一个单词组：Argument Marshelling 就是说将参数打包成一个结构体，然后传递给函数。（参数编组）

## Shared-Memory Hardware

### Cache Coherence(缓存一致性)

<img src="https://s2.loli.net/2024/05/21/RdksgVmJK1u3qwN.png" alt="image-20240519170721942" style="zoom:67%;" />

现在有一个x=3在主存里面，这里有多个处理器，处理器想要获取x的值，要把x的值加载到自己的缓存里面，这个时候就会出现缓存一致性的问题，比如处理器1把x的值加载到自己的缓存里面，然后处理器2也把x的值加载到自己的缓存里面，这个时候处理器1修改了x的值，这个时候处理器2的缓存里面的x的值就是过期的，这个时候就会出现缓存一致性的问题。

**值得注意的是，如果某个处理器想要获取这个x的值，它可以通过从其它处理器的缓存中获取，这样会更快一点。**

这里有一个问题在于，最后一个处理器修改了x的值，这个时候x的值在其它处理器的缓存里面是过期的，这个时候就需要处理器1去主存里面获取x的值，这个时候就会出现性能问题。因此，多核的第一个重要问题就是缓存一致性问题，确保多个处理器的缓存里面的数据是一致的。

这样，就有一个解决该问题的基本协议：**MSI协议**，其核心思想就是：每一个Cache line都是被标记为一种状态，这里有三种状态：

- M：Modified，表示这个Cache line是被修改的，这个时候这个Cache line是独占的，其它处理器不能访问这个Cache line。
- S：Shared，表示这个Cache line是共享的，这个时候这个Cache line是可以被其它处理器访问的。
- I：Invalid，表示这个Cache line是无效的，这个时候这个Cache line是无效的，其它处理器不能访问这个Cache line。 

目前Cache line的大小主要是64字节。

<img src="https://s2.loli.net/2024/05/21/1SUMbzQTXiWk3Zc.png" alt="image-20240520162000096" style="zoom:67%;" />

如果一个处理器想要更改y的值，比如第二个处理器想设置y=5，那么对应处理器的Cache line就会变成M状态，这个时候其它处理器的Cache line就会变成I状态，这个时候其它处理器就不能访问这个Cache line了。如下所示：

<img src="https://s2.loli.net/2024/05/21/vXwDUAEKTFjI8VH.png" alt="image-20240520162701888" style="zoom: 67%;" />





## Concurrency Platforms(并发平台)

> 为什么要有并发平台？因为多核编程是一个非常复杂的问题，我们需要一些工具来帮助我们解决这个问题，这就是并发平台的作用。一个并发的平台，抽象出了多个处理器之间的同步、通信协调以及提供负载均衡等等。

### Fibonacci Program 

```c
#include <inttypes.h>
#include <stdio.h>
#include <stdlib.h>

int fib(int n) {
    if (n <= 1) {
        return n;
    } else {
        int64_t x = fib(n-1);
        int64_t y = fib(n-2);
        return fib(n-1) + fib(n-2);
    }
}

int main(int argc, char *argv[]) {
    int64_t n = atoi(argv[1]);
    int64_t result = fib(n);
    printf("Fibonacci of %" PRId64 " is %" PRId64 ".\n, n, result);
    return 0;
}
```

这样的递归程序实际上是一个非常低效的程序，这个程序消耗的时间是指数级的。其实有更好的方法去计算斐波拉契数列，有一种方法使用线性的时间复杂度，即自下而上计算斐波那契数列/动态规划的方法。还有一种方法去计算斐波那契数列，花费log级别的时间复杂度，即基于平方矩阵的方法。

<img src="https://s2.loli.net/2024/05/21/isxLqrbgHo3R2pw.png" alt="image-20240520171701306" style="zoom:67%;" />

如果我们关注fib函数的计算过程，我们可以发现，上述图中，fib(2)被重复计算了多次。实际上并行计算的核心思想在于我们可以同时去计算fib的两个递归子调用，并且这种并行也是可以递归的。这样的并行计算的核心思想就是：**分治**。

### Pthreads

> 我们来看我们是怎么使用Pthreads去实现这个简单的斐波那契数列的。

Pthreads是一种标准的API，去处理多线程的问题。其被构建为一种C语言的库，其提供了一些函数去创建线程、同步线程、销毁线程等等。每一个线程都实现了处理器的抽象。然后这些资源被复用到实际的机器资源上面。

#### Key Pthread Functions

```c
int pthread_create(
	pthread_t *thread, 
	// 返回一个线程的标识符
    const pthread_attr_t *attr, 
    // 线程的属性
    void *(*start_routine) (void *), 
    // 线程的入口函数
    void *arg
    // 线程的参数，存储了创建线程的相关参数
);// 返回int类型的值，如果成功返回0，否则返回错误码

int pthread_join(
    pthread_t thread, 
    // 线程的标识符,该线程等待加入
    void **status
    // 线程的返回值
);// 返回int类型的值，如果成功返回0，否则返回错误码，这个函数的本质就是在我们继续执行主线程之前，等待子线程执行完毕。

```

接下来是关于用Pthread去实现斐波拉契数列：

```c
#include <inttypes.h>
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

int64_t fib(int n) {
    if (n <= 1) {
        return n;
    } else {
        int64_t x = fib(n-1);
        int64_t y = fib(n-2);
        return x+y;
    }
}

typedef struct{
	int64_t input;
	int64_t output;
}thread_args;

void *thread_func(void *ptr){
	int64_t i = ((thread_args *)ptr)->input;
    ((thread_args *)ptr)->output = fib(i);
    return NULL;
}// 这个函数的作用是通过线程去计算斐波拉契数列，其中ptr是一个指向thread_args的指针，这个结构体存储了线程的输入和输出。我们在实际计算的时候，类型转化为thread_args类型

int main(int argc, char *argv[]) {
    pthread_t thread;
    thread_args args;
    int status;
    int64_t result;
    
    if(argc<2){return 1;}
    int64_t n = strtoul(argv[1], NULL, 0);
    if(n<30){
        result = fib(n); // 如果n小于30，那么我们直接使用递归的方法去计算斐波拉契数列
    }
    else{
        args.input = n-1;
        //将输入参数编组到线程中去
        status = pthread_create(&thread, NULL, thread_func, (void*) &args);
        //调用线程函数，传入函数指针，即线程入口以及线程需要的参数，这里转化为void *类型是因为pthread_create函数的参数是void *类型的
        
        // 如果创建线程失败，那么我们直接返回，如果上述线程创建成功，将会返回NULL
        if(status){ // if status ！= NULL
            printf("Error creating thread.\n");
            return 1;
        }
        result = fib(n-2); // 这里的计算是和线程是同步并行的计算，即fib(n-2)和fib(n-1)是并行计算的
        
        status = pthread_join(thread, NULL);
		//获取线程的结束状态，这里的NULL是因为我们不关心线程的返回值
        if(status){
            printf("Error joining thread.\n");
            return 1;
        }
        result += args.output;
    }
    printf("Fibonacci of %" PRId64 " is %" PRId64 ".\n", n, result);
    return 0;
}
```

> 上述有一点要注意的是，关于指针函数，传入的参数为 Void * 的指针，其实这是一种泛型指针，可以指向任何类型的指针，这样的好处是可以传入任何类型的指针，但是在使用的时候需要进行类型转换。

上述代码中，一共进行了两个并行，也就是说，如果我有四个处理器，那么运行速度只有两倍，因为只有两个并行。这样的并行是有限制的，因为我们的这种并行只在顶层创建线程，并没有递归地去做。如果我想递归地创造线程，实际上代码会更加复杂。

### Threading Building Blocks

这是一个Intel公司开发的一个并行编程库，其提供了一些高级的抽象，比如并行循环、并行任务等等。其被实现为一个C++的库，其提供了一些类和函数去实现并行编程。这个库的核心思想是：**任务并行**。（编程人员具体是以任务为划分，而不是线程）这些任务使用工作窃取的算法去在线程之间自动进行负载均衡。

TBB更多的是关注性能，其写出来的代码要比Thread更简单，接下来我们来看用TBB写的斐波拉契数列的代码：

```c++
using namespace tbb;
class FibTask: public task{
public:
    const int64_t n;    // 这里的const表示这个变量是不可变的
    int64_t* const sum; // 这里的const表示这个指针是不可变的
    FibTask(int64_t n_, int64_t* sum_) :
    	n(n_), sum(sum_){} // 对const变量进行初始化
    
    task* execute(){
        if(n<2){
            *sum = n;
        }else{
            int64_t x, y;
            FibTask& a = *new(allocate_child()) FibTask(n-1, &x);
            // 这样的写法 new(allocate_child())后面接的是一个类的构造函数。
            // 这种写法实际上是在一个分配好的内存中构造一个对象
            FibTask& b = *new(allocate_child()) FibTask(n-2, &y);
            // 递归地去创建任务，这里的allocate_child()是一个函数，用来分配一个新的任务，同时&a和&b是引用，这样可以避免拷贝,因此要赋值*new(allocate_child())，这样才能保证a和b是引用
            set_ref_count(3);
            // 这里的set_ref_count(3)是设置任务的引用计数，这里的3表示两个子任务+任务本身，我们需要等待的任务数
            spawn(b); //启动任务b
            spawn_and_wait_for_all(a); // 启动任务a，并等待任务a、b都完成
            *sum = x+y;
        }
        return NULL;
    }
};
```

```c++
#include <cstdint>
#include <iostream>
#include <tbb/task.h>

int main(int argc, char *argv[]){
    int64_t res;
    if(argc<2){
        return 1;
    }
    int64_t n = 
		strtoul(argv[1], NULL, 0);
    FibTask& a = *new(task::allocate_root()) FibTask(n, &res);
    // 这里的new(task::allocate_root())是在根任务上面创建一个新的任务
    task::spawn_root_and_wait(a);
    // 这里的task::spawn_root_and_wait(a)是启动根任务，并等待根任务完成
    std::cout << "Fibonacci of " << n << " is " << res << std::endl;
    return 0;
}
```

用TBB写的代码能够获得更高的并行性。同时，TBB还有很多其它的性能：

- **并行循环**：tbb::parallel_for
- **并行减少**：tbb::parallel_reduce，用于数据聚合，比如你想要并行计算一个数组的和
- **管道和过滤器**：用于软件流水线的并行计算

同时，TBB还提供了各种并发的容器类，可以允许多个线程去安全地、并行地更新、修改容器中的item，比如有一个并发的队列、并发的哈希表等等。

### OpenMP

OpenMP实际上是一种业界标准，有多种编译器都支持OpenMP，比如GCC、Intel C++ Compiler、Clang等等。实际上，我们可以通过在代码中使用一些编译器编译指示来指定哪些部分代码是并行的，这样编译器就会自动地去并行化这些代码。OpenMP支持循环并行、任务并行以及流水线并行。

```c
// Fibonacci in OpenMP
int64_t fib(int n) {
    if (n <= 1) {
        return n;
    } else {
        int64_t x, y;
#pragma omp task shared(x,n)
        x = fib(n-1);
#pragma omp task shared(y,n)
        y = fib(n-2);
#pragma omp taskwait
        return x + y;
    }
}
```

上面代码中，# 开头的语句，我们称为Compiler directive 也叫编译器指令，用于创建并行任务的指示为: #pragma omp task。这样的指示告诉编译器，这个任务是并行的，编译器会自动地去并行化这个任务。这样的指示可以用于循环并行、任务并行以及流水线并行。上述代码中，对于shared这个关键字，表示两个变量在不同线程中是共享的。

### Cilk Plus

Cilk是C/C++的一种扩展，其提供了一些关键字去实现并行编程。Cilk Plus是Intel公司开发的一种并行编程的扩展，其提供了一些关键字去实现并行编程。Cilk Plus提供了一种更加高级的抽象，比如并行循环、并行任务等等。而对于Cilk Plus中的Plus部分，主要是支持矢量的并行。

```c
// cilk code for Fibonacci
int64_t fib(int n) {
    if (n <= 1) {
        return n;
    } else {
        int64_t x = cilk_spawn fib(n-1);
        int64_t y = fib(n-2);
        cilk_sync;
        return x + y;
    }
}
```

上述代码中，我们只使用了两个语句，一个cilk_spawn 和 cilk_sync。cilk_spawn表示其后面的子函数可以与父调用者并行进行，可以，cilk_sync表示所有的生成的子级返回之前，控制无法通过此节点。

> 值得注意的是，cilk_spawn只是一个提示，编译器可以选择是否并行执行这个任务，这样的好处是编译器可以根据实际情况去选择是否并行执行这个任务。实时运行系统可以根据实际情况去选择是否并行执行这个任务。

#### Loop Parallelism in Clik

<img src="https://s2.loli.net/2024/05/21/suCmJyMQlk4G7vw.png" alt="image-20240521143800762" style="zoom:67%;" />

现在我有一个任务，就是把矩阵对角线的数据进行交换，即做转置运算。这个时候，我们可以使用循环并行去实现这个任务。这个时候，我们可以使用Cilk Plus的关键字cilk_for去实现这个任务。

```c
cilk_for(int i=1; i<n; ++i){
    for(int j=0; j<i; ++j){
        double temp = A[i][j];
        A[i][j] = A[j][i];
        A[j][i] = temp;
    }
}
```

实际上，对于编译器来说，会去掉cilk_for，然后在内部使用嵌套(nested)的cilk_spwan和cilk_sync。也就是说，不同的迭代是并行的。

#### Reducers in Cilk（Cilk中的规约）

我们来看一个并行求和的例子：

```c
unsigned long sum = 0;
for(int i = 0; i<n; ++i){
    sum += i;
}
print("%d\n",sum);
```

首先，第一种最直接的方法就是尝试把for换成cilk_for，代码如下：

```c
unsigned long sum = 0;
cilk_for(int i = 0; i<n; ++i){
    sum += i;
}
print("%d\n",sum);
```

很明显，如果我们对于规约性质的工作也是这样去做并行优化，明显没有用，因为这里的每一次迭代是有关系的。甚至使用这样的cilk_for不一定会给你正确的答案。核心思想就是，这里存在：**确定性竞争**，即多个处理器可以写入同一内存位置。

下面是一个优化示例，使用了超对象（hyper object）以及Cilk中的规约器（Reducer），我们用一个宏去定义这个sum变量，其具有特殊的加法函数，我们通过宏来定义：

```c
CILK_C_REDUCER_OPADD(sum, unsigned long, 0);  // 通过宏构建具有加法函数的对象
CILK_C_REGISTER_REDUCER(sum);                 // 通过宏注册规约器

cilk_for(int i = 0; i<n; ++i){
    REDUCER_VIEW(sum) += i;                   // 通过这个VIEW的宏并行计算的结果与顺序的结果是相同的，Reducer会为我们处理这种确定性竞争
}
print("%d\n",REDUCER_VIEW(sum));
CILK_C_UNREGISTER_REDUCER(sum);               // 使用完这个规约器之后，再通过这个宏去告知已经完成使用该规约器
```

