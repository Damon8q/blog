# GO Runtime 简介
Go Runtime是Go语言运行所需的基础设施。其主要实现一下功能：
1. 谐程调度，内存分配，GC
2. 操作系统及CPU相关的操作的封装（如信号处理、系统调用、寄存器操作、原子操作等），CGO
3. pprof，trace，race检测的支持
4. map，chan，string等内置类型及反射的实现

与Java不同，Go没有虚拟机，Runtime被直接编译成native code；
Go Runtime与用户代码一起打包在一个可执行文件中，用户代码runtime代码在执行的时候没有本质区别，都是函数调用；
Go对系统调用的指令进行了封装，可不依赖glibc。
 m,n,       
一些go的关键字被编译器编译成runtime包下的函数：

| 关键字 | 函数 |
| ---- | ---- |
| go   | newproc | 
| new  | newobject |
| make | makeslice, makechan, makemap, ... | 
| <- -> | chansend1, chanrecv1 | 

# Goroutine
