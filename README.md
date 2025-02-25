# learn-systolic-array

#### 知识背景

矩阵加速器大致可以分为三种架构：

- 内积（inner product）
  
  即按向量内积逐个计算结果矩阵中的元素。通常是一排乘法器+一个加法树（adder tree）的结构，乘法器计算向量内积中的乘法，加法树对乘法的结果进行累加。一些语境里，加法树也被叫做reduction tree，因为把一串乘法结果累加成一个元素的操作可被视作一种reduction。
  
  典型例子：
  
  - 虽然NVDIA从未公开过其矩阵加速核tensor core的内部结构，不过从其调度粒度看tensor core很大概率是这种内积结构。
    
  - SIGMA（[SIGMA: A Sparse and Irregular GEMM Accelerator with Flexible Interconnects for DNN Training | IEEE Conference Publication | IEEE Xplore](https://ieeexplore.ieee.org/abstract/document/9065523/authors#authors)），一个对稀疏和不规则矩阵进行过额外优化的加速器架构，2020年HPCA的best paper。
    
- 外积（outer product）
  
  两个n\*1向量相乘，内积的结果是一个元素，外积的结果是一个n\*n的矩阵。用向量外积进行(n\*p) \* (p\*m)的矩阵计算，即通过外积先得到p个(n\*m)的子矩阵，再把这p个子矩阵累加在一起得到最终的结果矩阵。
  
  显然这是一种比较不方便的架构（要存储并累加大量结果矩阵）。这种架构通常不用于稠密矩阵的乘法，而是用在稀疏矩阵（尤其稀疏度较高的矩阵）乘法里。
  
  例子：
  
  - Sparch（[SpArch: Efficient Architecture for Sparse Matrix Multiplication | IEEE Conference Publication | IEEE Xplore](https://ieeexplore.ieee.org/abstract/document/9065428)），一个基于外积架构的稀疏矩阵加速器。
- 脉动阵列（systolic array）
  
  用2D阵列计算矩阵乘法，典型例子是谷歌的TPU（[In-Datacenter Performance Analysis of a Tensor Processing Unit | Proceedings of the 44th Annual International Symposium on Computer Architecture](https://dl.acm.org/doi/abs/10.1145/3079856.3080246)）。
  
  这种设计的好处是数据复用率非常高，对内存带宽的要求较小，能缓解大规模矩阵计算的“内存墙”问题，并减小内存读取产生的功耗；弊处是灵活度差，处理不规则或稀疏矩阵的效果不太好。
  

这三种架构里常用的是内积和脉动阵列，前者类似灵活的小核，后者类似不灵活但在数据量大时整体效率更好的大核。至于外积架构用的比较少，大多用在剑走偏锋的稀疏加速器里。

#### 上手学习

0. 大概率会用到的基本技能树如下：
  
  - verilog/system verilog，最好会写流水线乘法器、简单的流水线riscv处理器、FIFO
    
  - FPGA的workflow。需要在vivado里看时序做优化。
    
  - ASIC的workflow。可以试着跑通OpenROAD，一个开源的ASIC设计工具链。
    

1. 用RTL手动写一个简单的脉动阵列。
  
  一个简单的脉动阵列需要包括：
  
  - 阵列本身
    
  - 输入模块。输入给阵列的是一个倾斜的矩阵，需要一个模块把矩阵斜过来
    
  - 输出模块。把计算结果从阵列里读出来
    
  
  以上的脉动阵列已经可以完成矩阵计算。如果要用来计算卷积，还需要将卷积转换为通用矩阵乘法（GeMM）的模块：
  
  - 硬件im2col模块。
    
  
  关于RTL脉动阵列的参考资料蛮多，可自行在github/bilibili上搜索脉动阵列、TPU等关键词。
  
2. 有一些经典论文可看。
  
  首先是脉动阵列的分块。通过分块提高脉动阵列的灵活性和实际利用率是个非常常见的优化。以下三篇在分块的思路上大同小异，可以挑一篇细读。
  
  Scale-out Systolic Array [Scale-out Systolic Arrays | ACM Transactions on Architecture and Code Optimization](https://dl.acm.org/doi/full/10.1145/3572917)
  
  ReSA: Reconfigurable Systolic Array for Multiple Tiny DNN Tensors（[ReSA: Reconfigurable Systolic Array for Multiple Tiny DNN Tensors | ACM Transactions on Architecture and Code Optimization](https://dl.acm.org/doi/full/10.1145/3653363)）
  
  FlexSA: Flexible Systolic Array Architecture for Efficient Pruned DNN Model Training[[2004.13027] FlexSA: Flexible Systolic Array Architecture for Efficient Pruned DNN Model Training](https://arxiv.org/abs/2004.13027)
  
  其次是在脉动阵列上做稀疏矩阵的加速。一下两篇体现了两个比较常见的作法：要么是对稀疏数据进行打包预处理，要么是较大幅度魔改阵列设计、直接支持稀疏。还有一些设计是专门支持结构化稀疏的，但结构化稀疏的硬件支持太简单了，不太构成一个科研问题。
  
  STPU [Sparse-TPU | Proceedings of the 34th ACM International Conference on Supercomputing](https://dl.acm.org/doi/abs/10.1145/3392717.3392751)
  
  DSSA [DSSA: Dual-Side Sparse Systolic Array Architecture for Accelerating Convolutional Neural Network Training | Proceedings of the 51st International Conference on Parallel Processing](https://dl.acm.org/doi/abs/10.1145/3545008.3545086)
  
3. 通常来说，如果你具备初步的RTL代码经验，一周时间应该足够写出一个简单的RTL脉动阵列（写个4*4的就够了，这个肯定比五周期流水线risc-v好写、好测一些）。两周时间应该足够结合自己的代码和论文对脉动阵列的性质有个基本了解，在此基础上可以试着思考这种架构仍然有哪些有待解决的不足，以及可能用什么方法解决。
