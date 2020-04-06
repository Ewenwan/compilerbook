# compilerbook

现代体系结构的优化编译器.pdf

高级编译器设计与实现.pdf

> 加快计算速度发展

单机单发射单级流水线 ---> 多发射多级流水线(并行取指+并行执行指令)---> 再+向量化(数据并行，向量寄存器执行部件)---> 多线程 ---> 分布式多机器

编译器 应该能 自动执行 上述 优化pass，能够达到或超过 手工优化的方式才够格。

指令调度，重排指令序列(避免数据相关的前后指令依赖、控制相关的分支预测失败等)，使得指令流水线尽量不出现流水线停顿(pipeline stall),使得流水线保持充满状态。

多发射：每个指令周期发射多条流水线指令，还有得可以乱序发射指令。

存储系统相关，时延:从存储器传送单个数据需要的处理器周期数；带宽:每个周期从存储系统能传送到处理器的数据元素个数。

避免时延:经常使用(引用)的值存放在快速中间存储器(处理器中的寄存器，cache高速缓存)，第一次引用后，之后的引用代价很低。

延迟容忍:在取数据时做一些延迟容忍的其他事情，如**数据显示预取prefetching**、无阻塞取数。

循环变换，嵌套循环应该考虑内循环分段（循环分块），使其大小适合目标计算机的cache高速缓存的大小，这种机器各异的优化应当由编译器进行优化。

优化策略：寄存器分配、指令调度、减少数组地址计算开销，在串行代码中寻找并行性并定制到目标机器上，考虑机器的存储结构，寻找数据分解，执行并行操作。

依赖关系，顺序语句中的数据依赖(前一句的左操作数 是 后一句的右操作数)，分支语句中的控制依赖（有分支预测来部分解决，也可以通过**if转换**技术 转换为 数据依赖）

数据依赖:

**存-取依赖分类**

> 1. 真依赖     写后读 同一个操作数 WAR
```c
语句s1:  x   = ...   前一句的左操作数    写数据
语句s2:  ... = x   是后一句的右操作数    读数据  或交换s2的值可能会改变
```

> 2. 反依赖     读后写 同一个操作数 RAW
```c
语句s1:  ... = x     前一句的右操作数    读数据  若交换 s1的值可能会改变
语句s2:  x   = ... 是后一句的左操作数    写数据
```

> 3. 输出依赖   写后写 同一个操作数 WAW
```c
语句s1:  x = ...
语句s2:  x = ...  若交换可能会引起后面的语句读入X值出现错误
```

**循环内的依赖**

前后两个迭代实例中有上述 存-取依赖 中的一种，可以使用多面体模型来解耦这种依赖关系。

具体的为使用仿射变换来改变这种依赖关系变为依赖无关的，然后可以并行执行，向量执行。


循环交换，通常用于解决，紧邻两层循环中，一层有数据依赖，一层无数据依赖，将内存循环交换为无依赖的，然后将内层循环向量化。

```c
for I = 1, N
  for J = 1, M
     A(I, J + 1) = A(I, J) +  B  // J维度 有依赖关系 上一次的结果，下一次会用，写后读依赖
  end
end

// 可以将依赖关系的维度 交换到外层循环，内存循环就没有依赖关系了，然后就可以向量化执行了
for J = 1, M
  for I = 1, N
     A(I, J + 1) = A(I, J) +  B  // 内层循环无依赖关系
  end
end

// 内层循环可以向量化执行
for J = 1, M
     A(I:N, J + 1) = A(I:N, J) +  B  // 内层循环无依赖关系
end


```



**if转换**

删除程序中所有分支的过程，使用其他分支替换。

> 分支分类:

前向分支:控制转向到同一嵌套循环中本分支之后的位置(程序前方)；

后向分支:控制转向到同一嵌套循环中本分支之前的位置(程序后方)；

出口分支:结束一个或多个循环，将控制转向循环嵌套之外的位置；

