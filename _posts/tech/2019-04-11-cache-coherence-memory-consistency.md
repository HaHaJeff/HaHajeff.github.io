---
?layout: post
title: cache coherence和memory consistency
category: 技术
tags: [cahce]
keywords: cache, 体系结构
---

---

## consistency vs coherence
SWMR(Single-Writer-Multiple-Reader) invariant确保 **在同一时间对同一个内存地址，a)一个core可能写(读)这个地址，b)一个或多个core读这个地址**，Data-value invariant确保对该内存位置的更新可以被正确传送，也就是说core的cache总是拥有该地址的最新数据，故有cache coherence说法，cache间通过**协调**保证cache对一个值的确定。所以，为什么paxos叫做共识性算法而不是应该叫做一致性算法，这就是理由吧。各个节点间达成共识，对同一个值得读取保证得到最新得版本。

**总而言之：**

- cache coherence不等价于memory consistency
- memory consistency需要用到cache coherence，但仅仅只是使用，memory consistency不需要了解cache coherence，就像黑盒一样使用它

## memory consistency

- 定义：多线程程序对**共享**内存操作的规范，不涉及cache coherence——"Consistency models define correct shared memory behavior in terms of loads and stores (memory reads and writes), without reference to caches or coherence."

## Sequence consistency
lamport老爷子对SC的定义：
- 在single processor中，SC的定义：执行结果与按照程序order执行一致 **the result of an execution is the same as if the operations had been executed in the order specified by the program**
- 在multiple processor中，SC的定义：任意一次的执行结果与所有processor按照某种顺序执行的结果一致，并且每一个独立的processor中的operations与程序的order一样

| core1  | core2  |
|--|--|
| S1: store data = new | L1: load r1 = flag |
| S2: store flag = set    | B1: if r1 != set goto L1 |
|     | L2: r2 = data |

**在SC模型中，core1 core2按照指令顺序执行两段程序，并且L2将作为最后一条指令执行，读到结果r2=new**

**requirements**

- L(a) <p L(b)      ====>   L(a) <m L(b)             load load can't reordering
- S(a) <p S(b)     ====>   S(a) <m S(b)             store store can't reodering
- L(a) <p S(b)     ====>   L(a) <m S(b)              load stote can't reodering
- S(a) <p L(b)     ====>   S(a) <m L(b)              store load can't reodering

**X86架构下CPU默认coherance语义：除了Store load不支持其余全部支持**

## complier reorder

编译器为了提高程序运行效率，会对指令进行重排

实验代码如下：

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/cache_coherence/compile_reorder_src.png)

使用

> gcc -S -masm=intel test.c

进行编译：

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/cache_coherence/compile_reorder_01.png)

使用

>  gcc-O2 -S -masm=intel test.c

进行编译：

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/cache_coherence/compile_reorder_02.png)

发现指令已经被重排

为了避免编译重排，采用汇编指令

> __asm__ __volatile__ ("" ::: "memory");

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/cache_coherence/compile_reorder_src_01.png)

使用

>  gcc-O2 -S -masm=intel test.c

进行编译：

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/cache_coherence/compile_reorder_03.png)

编译器memory barrier只是一个标识，在汇编代码中不会出现

## CPU reorder

解决了编译器层面的reorder，还有更加令人头疼的CPU reorder，CPU为了减少指令执行的延迟，会对执行的指令进行reorder

- load load reorder
- load store reorder
- store store reorder
- **store load reorder**

X86架构下，除了store load reorder其它三种reorder都不会发生

### CPU memory barrier

**为什么会重排？**

- 现代编译器的代码优化和编译器指令重排可能会影响到指令的执行顺序
- 指令执行的乱序优化、流水线、乱序执行、分支预测都可能影响到执行顺序

当然，这些重排也会保证不影响语义，可惜的是：**只保证在单核内的正确语义**

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/cache_coherence/arch.png)

**为什么需要CPU memory barrier？**

传统的mesi协议中开销最大的两个部分为：

- 将某个cache line标记为invalid状态
- 当某个cache line当前状态为invalid时写入新的数据。所以CPU设计者通过添加Store buffer和invalid queue加速这些操作

引入Store buffer和Invalid queue后，读写操作：

- 当一个核心在Invalid状态进行写入时，首先会给其它CPU核发送Invalid消息，然后把当前写入的数据写入到Store Buffer中。然后异步在某个时刻真正的写入到Cache Line中。当前CPU核如果要读Cache Line中的数据，需要先扫描Store Buffer之后再读取Cache Line（Store-Buffer Forwarding）。但是此时其它CPU核是看不到当前核的Store Buffer中的数据的，要等到Store Buffer中的数据被刷到了Cache Line之后才会触发失效操作。
- 而当一个CPU核收到Invalid消息时，会把消息写入自身的Invalidate Queue中，随后异步将其设为Invalid状态。和Store Buffer不同的是，当前CPU核心使用Cache时并不扫描Invalidate Queue部分，所以可能会有极短时间的**脏读**问题。这里的Store Buffer和Invalidate Queue的说法是针对一般的SMP架构来说的，不涉及具体架构。

**Intel提供了三种内存屏障指令**

- sfence，实现Store Barrier，会将store buffer中的数据刷入L1 cache中，使得其他cpu核可以观察到这些修改，而且之后的**写操作**不会提前到sfence之前，即sfence之前的操作一定在sfence完成之前全局可见
- lfence，实现Load Barrier，会将invalidate queue失效，并读取进L1 cache中，而且之后的读操作不会调度到lfence之前(并未规定全局可见)
- mfence，实现Full Barrier，同时刷新store buffer和invalidate queue，保证了mfence前后的读写操作顺序，同时要求mfence之后的写操作全局可见之前，mfence之前的写操作全局可见

**四种基本的CPU barrier**

- LoadLoad Barrier
- LoadStore Barrier
- StoreStore Barrier
- StoreLoad Barrier

**更为复杂的Barrier**

- Store Barrier(Write Barrier)
  - 所有Store Barrier**之前**的Store操作必须在Store Barrier之前完成，所有Store Barrier**之后**的Store操作必须在Store Barrier之后才能开始
  - Store Barrier只是针对Store操作，对Load操作无效
- Load Barrier(Read Barrier)
  - 将Store Barrier的全部功能转换为Load即可
- Full Barrier
  - Load + Store Barrier，Full Barrier两边的任何操作都无法交换

### CPU Memory Barrier使用

```
#define lfence() __asm__ __volatile__("lfence": : :"memory") 
#define sfence() __asm__ __volatile__("sfence": : :"memory") 
#define mfence() __asm__ __volatile__("mfence": : :"memory") 
```

---

[1] A Primer on Memory Consistency and Cache Coherence

[2] CPU Cache Coherence & Memory Consistency(何登成)

[3] [浅谈Memory Reordering](http://dreamrunner.org/blog/2014/06/28/qian-tan-memory-reordering/)
