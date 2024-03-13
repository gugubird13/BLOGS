---
title: MPI学习路线
date: 2024-03-13 21:31:23
tags: MPI
categories: learning
top_img: blue
---

# **HPC 学习资源**

开放编辑 😡

# **课程**

​                ● [MIT 6.172: Performance Engineering Of Software Systems](https://ocw.mit.edu/courses/6-172-performance-engineering-of-software-systems-fall-2018/)（性能优化）



# **书籍**

​                ● [Algorithms for Modern Hardware](https://en.algorithmica.org/hpc/)（体系结构）

​                ● [Programming Massively Parallel Processors: A Hands-on Approach](https://book.douban.com/subject/4265432/)（CUDA）

​                ○ 建议看英文原版，PDF 见群文件

​                ● [现代CPU性能分析与优化](https://book.douban.com/subject/36243215)（性能调优）

​                ● [MPI 教程](https://mpitutorial.com/tutorials/mpi-introduction/zh_cn/)（分布式计算、MPI）



# **论文 / 博客**

​                ● GEMM 性能优化（参考文献也可以看看）

​                ○ [Anatomy of High-Performance Matrix Multiplication](https://www.cs.utexas.edu/users/flame/pubs/GotoTOMS_final.pdf)（单线程）

​                ○ [Anatomy of High-Performance Many-Threaded Matrix Multiplication](https://www.cs.utexas.edu/~flame/pubs/blis3_ipdps14.pdf)（多线程）

​                ○ [Matrix multiplication on batches of small matrices in half and half-complex precisions](https://par.nsf.gov/servlets/purl/10190584)（batch_gemm）

​                ● [浮点数误差入门](https://zhuanlan.zhihu.com/p/673320830)（菜鸡群主写的）

​                ● [OneFlow/FasterTransformer SoftMax CUDA Kernel 实现学习](https://github.com/BBuf/how-to-optim-algorithm-in-cuda/tree/master/softmax)（CUDA）

​                ● [MPI与并行计算系列](https://zhuanlan.zhihu.com/p/35565250)（分布式计算、MPI）

​                ● [集合求并的极致优化](https://zhuanlan.zhihu.com/p/674886045)（菜鸡群主写的）



# **面经**

​                ● [AI/HPC面试问题整理](https://zhuanlan.zhihu.com/p/663917237)

​                ● [推理部署工程师面试题库](https://zhuanlan.zhihu.com/p/673046520)                             


（技能树）群主只会CPU上的ML推理，只知道这么点，实际上知识点有很多

**通用技能**

​                ● 体系结构：汇编、SIMD、Profiling、内存、……

​                ● 并行 / 并发：OpenMP、分布式计算、MPI

​                ● 加速器：GPU、NPU

​                ● 计算库：BLAS

**ML相关**

​                ● 框架 / 库：PyTorch、……太多了😭

​                ● 算子：GEMM、量化、算子融合、……

​                ● 并行算法：3个维度（TP、DP、PP）
