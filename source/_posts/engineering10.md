---
title: 性能优化学习笔记10
date: 2024-06-06 13:41:12
tags: performance_engineering
categories: learning
description: 性能优化学习笔记
keywords: Measurement, Timing
top_img: https://s2.loli.net/2024/03/14/WQxHO96fkU5RJSe.jpg
cover: 
comments: false
---
# 度量与计时

> 为什么没有第9课？因为感觉第9课没有什么记录的必要，因为实际上第九课告诉我们的是，编译器能做什么以及不能做什么，从四个角度告诉了我们编译器做的优化：优化标量、优化结构体、优化函数调用、优化循环。前两个主要是说的是我们不需要对参数先store再load，而是直接在寄存器中进行操作，后两个主要是说的是我们可以对函数调用进行内联优化，以及对循环中的有些变量可以提出来，减少循环次数。这些优化都是编译器可以做的，但是我们不需要关心，因为编译器会帮我们做好。
>
> 同时，最后还讲了编译器在某些情况下，比如内存重叠的情况下(Memory Aliasing)不能给数据做vectorize，因此我们一般要加个修饰符`restrict`，告诉编译器这个数据不会重叠，可以进行vectorize。

## 写在前面

To Mearsure Is To Know. 我们进行度量是为了更好的优化，而度量的目的是为了知道我们的程序的性能瓶颈在哪里，我们应该如何去优化。

## Case Study

我们来看一个对排序进行时间度量的代码：

```c
#include <stdio.h>
#include <time.h>

void my_sort(double *A, int n);
void fill(double *A, int n);

struct timespec start, end;

int main() {
    int max = 4*1000*1000;
    int min = 1;
    int step = 20 * 1000;
    double A[max];
    
    for(int n = min; n < max; n+=step){
        fill(A, n);
        
        clock_gettime(CLOCK_MONOTONIC, &start);
        my_sort(A, n);
        clock_gettime(CLOCK_MONOTONIC, &end);
        
        double tdiff = (end.tv_sec - start.tv_sec) + (end.tv_nsec - start.tv_nsec) / 1e9;
        printf("size %d, time %f\n", n, tdiff);
    }
    return 0;
}
```

上述代码主要是对不同大小的数组进行排序，并且度量排序的时间。按理来说，如果这个排序是合并排序，其绘制出来的曲线即n-time的曲线应该是符合O(nlogn)的，但是实际上并不是。

<img src="https://s2.loli.net/2024/06/06/UfKJqDt8VGw1mNk.png" alt="image-20240606170140501" style="zoom:67%;" />

实际上我们可以看到，这里的实际绘制曲线是一个规律的上升的连续像过山车一样的线，为什么会出现这个情况呢？

一个难以想到的结果是：**这个和时钟频率有关，随着程序的运行，CPU的频率会发生变化，这个变化是由于CPU的热量导致的，过热导致时钟频率下降，因此我们的时间度量是不准确的。**

> by the way，为什么上述抛物线越来越近？因为排序的时间会越来越长，这就使得机器的时钟频率变化基于相同的n间隔下，n越大，变化越频繁。

## DVFS

上述的这种情况，我们称为Dynamic Voltage and Frequency Scaling，即动态电压和频率调节。这种调节是由于CPU的热量导致的，过热导致时钟频率下降，因此我们的时间度量是不准确的。
$$
Power \propto CV^{2}f
$$
我们可以发现，如果减少电压和时钟频率，那么功率就减少，这样温度就降低下来了。

## QUIESCING SYSTEMS（静止系统）

> 如果你能够减少variability，那么你就能弥补系统以及随机度量上面的误差

### 导致Variability的来源有哪些？

- 守护进程与后台作业
- 系统中断
- 代码与数据对齐是否对齐
- 线程的放置——很多电脑系统喜欢用核心0来做很多事情，这样会导致核心0的负载很高，而其他核心的负载很低，这样会导致时钟频率的变化，从而导致variability。
- 实时调度器，这个调度器会导致variability，因为它会在不同的核心上运行不同的线程，这样会导致时钟频率的变化
- 超线程，也称为同步多线程，也就是一个执行单元，同时去运行两个指令流，这样会使得性能获得加速，我们一般在度量的时候，会关掉这个
- DVFS，就是之前讲的那个

### 什么是静止系统？

静止系统就是我们在度量的时候，尽量减少variability，使得我们的度量更加准确。那么我们应该如何去做到这个静止系统呢？

- 确保没有其它工作在运行
- 关掉守护进程与后台作业
- 断网
- 不要瞎动鼠标
- 对于线性工作，不要在core0上运行，因为通常中断处理器在那个上面运行
- 关掉超线程(hyperthreading)
- 关掉DVFS

> 现在有一个问题，如果我有一个串行的程序，我们设置了静止系统，但是我们的程序还是没有达到我们预期的性能，这是为什么呢？
>
> 这个答案与内存有关，内存错误是不确定的，当我们访问DRAM的时候，这里的粒子如果发生碰撞并且使得比特位翻转了，那么这个时候硬件就要处理，但是需要一个额外的周期去完成它，因此我们说内存错误是不确定的。

### Code Alignment导致的Variability

比如我在代码里面插入了一个字节的内容，这可能导致有些内容会跨越页(Page)边界，这样会导致内存访问的时间变长，因此我们的度量就会变得不准确。

> 值得一提的是，在我们执行编译命令行的时候，我们在命令行指定的.o文件的顺序也是有讲究的，很神奇吧！
>
> 甚至可执行文件名也会影响！！！

## 度量软件性能的工具

1. 从程序外部去度量，其中包括三个指标，分别是：Wall Time、User Time、System Time，Wall Time是程序从开始到结束的时间，User Time是程序在用户态的时间，System Time是程序在内核态的时间。Wall Time ≠ User Time + System Time，因为Wall Time还包括了一些其他的时间，比如IO时间等。
2. Instrument，这个词我第一次在[Algorithms for Modern Hardware](https://en.algorithmica.org/hpc/)里面看到，这个中文翻译叫检测，也就是在程序中插入一些代码，来度量程序的性能，但是这个方法有一个问题，就是会改变程序的性能，因此我们要尽量减少这种影响。
3. 使用中断，我们停止程序，然后去看它的内部状态
4. 使用硬件或者操作系统的支持，比如使用硬件计数器来看程序的性能，但是这个问题就在于counter得到的是周期数，你要把周期数转化为时间，这个还是比较复杂的
5. 使用模拟器，simulator

## 性能建模

<img src="https://s2.loli.net/2024/06/06/3HkSF9WTJECOKfQ.png" alt="image-20240606205106337" style="zoom:67%;" />

上述是基本的性能工程的工作流，我们主要就是不断去使得优化过后的程序打败原来的程序，这个过程就是性能建模。

因此，我们在度量两个程序的性能的时候，尽管会有很多的Variability，但是我们核心还是去做rank count，看哪个程序打败对方更多，那么这个程序就win了，也有人喜欢把两个程序的10% low的时间去做对比，但是这样做统计意义上是没有依据的。