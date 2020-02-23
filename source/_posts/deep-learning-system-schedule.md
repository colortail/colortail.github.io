---
title: 深度学习系统知识结构
date: 2019-05-08 21:08:07
tags: deep learning
mathjax: true
---

1. 常见的深度学习模型

   1.1 CNN，用于图像处理，网络有三个基本层：

   convolutional layer：包括用卷积核的计算,然后用激活函数（如Relu）计算

   pooling layer：滑动窗口来降采样，MaxPooling取窗口最大值，AveragePooling取窗口内数据的平均值。然后再补0，保证图片一样大小。

   Flatten layer：用来叠加多种卷积核计算的结果，最终结果接到全连接层上，用softmax做多目标分类。

   

   1.2 GAN，生成对抗网络，用于真实数据分布的生成，比如图像，视频。

   模型包括一个生成模型和判别模型，生成关于真实数据的分布。判别模型会最大化自己的准确度，而生成模型会最小化判别模型的准确度。两者交替优化。

   

   1.3 RNN/LSTM，RNN具有反馈结构，用来处理序列类的问题，如语音等。

   

   1.4 BERT，用于处理NLP中机器理解，翻译类的问题。用无监督的方法训练transformer模型。是一个深且窄的模型。会同时利用左侧和右侧的词语，被称作Masked（遮挡）语言模型。训练参数比transformer要少，不需要海量的语料。

   

   1.5 Transformer，也是NLP上处理翻译问题的。Seq2Seq结合了Attention和RNN，Transformer就是全Attention，和Seq2Seq一样，包括Encoder和Decoder。Attention就是在decoding阶段对input的信息赋予了不同的权重。

   

2. 后向传播算法（BP），矩阵求导，常见算子的梯度推导（矩阵乘，卷积，池化，Relu），batch normalization

   

3. autograd的基本原理。是pytorch里用来计算梯度的

   

4. cuda编程，cuda高阶用法

   stream指按照host调用的顺序执行的一堆异步的操作，event用来标记stream执行的特定的点，用于同步和操控运行步调。

   异步/同步指的是host提交任务到device后，host是阻塞还是继续执行。kernel函数总是异步，cudaMemcpy总是同步。优化常见cuda kernel：element-wise product 元素积，reduce，broadcast，MatMul， conv，pooling

   

5. C++，Python。设计模式，惯用法，模版；vim，gdb调试

   

6. socket，RDMA编程（Remote Direct Memory Access）是专门的网卡和InfiniBand使用的。

   常见collective operation代价分析：ring allreduce：对于某一块数据，先scatter reduce让一个gpu上有全部的数据，再做all-gather传递这个全部的数据。tree all reduce代价分析（都在gpu上）

   collective operation是指数据被同时发送到多个节点，或者从多个节点接收。包括：gather：从所有节点收集数据，scatter：其中一组数据被分解成片段，并且不同的片段被发送到所有的节点，broadcast：相同数据被发送到所有节点。

   

7.  多线程编程，熟悉锁，条件变量，内核线程，用户级线程，对actor，CSP（Goroutine + Channel）各种技术熟悉。

   

8. 熟悉编译器的基本原理，dataflow分析，多重循环程序优化技巧，典型的是polyhedral模型

   

9. 常见分布式系统原理，map reduce，spark， flink， tensorflow

   

10. 计算机体系结构，量化分析方法，

    amdahl' law , 1/((1-P)+(P/N) ：P是程序并行化比例，N是处理器个数

    Roofline model

    流水线分析

    

11. 操作系统原理及常用系统诊断工具，资源利用率分析