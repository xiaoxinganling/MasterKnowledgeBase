## Broom: sweeping out Garbage Collector from Big Data Systems

### 简要总结

​	Broom 是发表在 HotOS'15 上篇幅较为短小的一篇文章，全文只有 5 页。

​	大数据处理系统使用基于high-level language，例如 Java，C# 等，的**自动内存管理方案**管理任务执行所需的内存，而**大量对象的产生**以及**更大size的堆空间**（heap）都为GC带来了巨大压力。本文的核心思想是使用region-based memory management管理大数据处理任务产生的内存，如此一来，对象被分配在一系列region中（region中的对象统一被回收），其被回收时也无需遍历整个heap。

### 背景介绍

​	JVM和 .NET CLR为大数处理任务带来了巨大的好处——强类型语言、自动内存管理方案以及高阶函数等，然而这些优点需要一定的代价，长时间的GC暂停降低了大数据处理任务的吞吐量，同时提高了其时延。考虑到大数据处理系统在使用内存时有以下两个特点（**作者观察到的现象**）：

- 任务的执行过程均显式或隐式地基于图，而这些图（graph）由一系列 **stateful** data-flow operator组成，它们分别由不同的线程执行；同时，这些operator大多基于事件（event-based）进行通信或数据传输，因此它们相对独立（MapReduce中的map、reduce function，Spark在每个task上执行一个action）
- 这些operator产生的对象在operator结束后大多消亡，并且在逻辑上能够以batch（分批）的形式进行分组（batch之间的对象在回收时互不影响）（Naiad中对象根据其key和timestamp进行存放）

​	考虑到同一个batch中的对象之间存在较少的隐式关联，而在大数据处理系统中用户通常只向system-defined operators提供相应的代码片段，因此，对system-defined operators做出修改可以做到对用户透明。于是，作者考虑在不要求用户添加额外注解的情况下，实现region-based memory management——Broom（基于Naiad）。

### 解题思路

​	既然是region-based memory management，则应该考虑如何分配内存、回收内存以及处理跨region的引用。

#### 分配与回收内存

​	对于使用communicating actor（使用消息进行通信或数据传输，事件驱动）的大数据处理系统而言，作者阐述了三种region：

- transferable region：used for message（主要用于actor之间的消息传输），可以存活在多个actor中，但是只能被作为持有者的actor访问
- actor-scoped region：region中的对象和actor的生命周期相同
- temporary region：存放临时数据的region，这些临时数据很少通过message进行传递或者保存

​	作者修改了诸如Aggregate等actor的代码，并将代码中分配内存的部分全部重定向到相应的region中，三种region中内存的释放策略有所不同，temporary region的生命周期最短，在一个actor中随时都可以释放；而actor-scoped region的生命周期与actor的生命周期相同；transfer region的生命周期文中没有详细说明，但通过本文提供的例子可大致判断只有当transfer region中的数据被处理完毕，不再被需要时即可释放其占用的内存，这与特定的actor相关。

#### 跨region的引用处理

​	限制位于不同region中对象之间的指针指向关系来实现memory safety，但具体还没实现，详细操作见论文 *p4*，此处不再赘述。

### 启发与思考

​	这篇文章的思路和我们的研究工作很像，其关键点也在于不需要用户的额外开销即可优化大数据处理系统的内存使用。本文的实现思路是region-based memory management，选择的大数据处理系统是Naiad，优化手段也和Naiad中的各种Actor密切相关。文章的future work希望通过静态分析等手段减少 system developer（不是上层用户，是底层系统工作者）的工作量，但查阅了相关资料后并没有看到其具体实现。总体来说，该工作与我们的研究有大量相似点（都想实现对用户透明，都专注于大数据处理系统的内存模式分析），但我们的工作不局限于region-based memory management，也可能会引入诸如[MEMTUNE](./memtune_con.md)的优化手段；其次，我们专注于Spark中的iterative/shuffle-intensive application，并对其进行优化，而作者只是针对特定几个应用进行了优化，并不算一个完整的工作。本文的启发如下：

- 正如本文所说，如果注解添加在用户看不见的地方，例如system-provided structure等，优化方案就不会为用户带来额外成本，这为我们的研究工作提供了一定的指导作用；此外，静态分析的手段能够显著降低我们（系统工作者，区别于上层的用户）的工作量，可以考虑将其作为优化手段，但本文的future work一直没有实现，因此我们可以借鉴**静态分析**，但不可陷入过深，太专注于这项技术（指静态分析）；
- 对于那些**一次回收大量对象**的应用，region-based memory management能够取得较好的性能表现，考虑到我们针对的应用特性，最后肯定绕不开region-based memory management这一关键点，如何在此基础上**分析出新的内存模式**，**发现具体场景下实际存在的问题**是研究的关键；
- 对于那些传统的应用，我们的工作是利用大数据处理系统提供的domain-specific knowledge优化相应应用中GC的使用；而对于已有的部分工作（以POLM2和Deca为例），我们的目标是减少用户所需的额外开销（对用户透明），提高其实用性；而对于另一部分无需用户添加注解的应用（如本文提到的Broom），我们的工作的不同之处在于**针对某一类应用而不是某些特定应用**挖掘出内存模式并完善与本文类似的内存管理方案（1.Broom针对的是某些特定的应用，我们想往上拓展；2.涉及的背景大体相同，但**看待问题的角度不同^1^**，最终解决方案**可能**不同）

^1^ 例如，我们针对的是Spark，Spark与Naiad在内存模式上肯定存在不同之处（有很多相同之处）；此外，尽管背景大体相同，但如果看问题的角度不同，最后要解决的问题和其解决方案肯定有所不同（这些文章都想优化大数据处理系统的内存使用，而且大多数paper都阐述了GC的劣势并想再此之上做文章，但他们针对不同的角度给出了不同的问题理解以及不同的解决方案）