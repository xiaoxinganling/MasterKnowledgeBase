# Yak 论文总结

## 一句话总结（主要干了些啥）

​	这篇系统文章深入浅出，从背景分析到问题提出以及实现细节的许多思想都十分详实，因此，第二次读这篇文章还是能收获到很多东西。废话少说，直接总结一下这篇文章。

​	首先我们知道，大数据分析系统里的 GC 不给力这件事已经在很多篇文章中提及，并且众多相关实验都给出了一个不争的事实：GC 会拖慢程序的执行，使程序执行的总时间提升；除此之外，GC 还会导致不可避免的暂停，总而给“延迟敏感”的应用程序带来了难以接受的延迟。

​	在这样的背景下，我们分析一下原因到底是什么。数据量的增加是一个原因，但核心问题还是在于大数据分析系统里对象的特性和GC的应用场景很不一样。

​	GC 在设计时就假设应用程序里的大部分对象都是 temporary 的，过一段时间后这些对象就会挂掉。因此，我们可以看到 JVM 的堆被分成了两部分，即小一点的年轻代，大一点的老年代。首先我们来解释为什么分成两部分：这是因为我们使用了基于 copy 的 gc 算法，即我们只使用所有内存的一部分（可以是一半）为新创建的对象提供内存，当这部分内存满了，我们将存活的对象拷贝至另一部分，然后这部分内存又可以作为一块新的内存为新创建的对象提供内存了，这样做的好处就是不会产生内存碎片，而且算法的设计也很方便（**暂时这么理解，如果想改的话再改**）[^1]

​	又因为相关研究发现，当进行内存回收时，有百分之九十以上的对象都是 dead 的，所以我们会将年轻代和老年代设置为一小一大。这样的好处是我们不用每次回收对象时都 scan 整个 heap，这样可以节省时间。而且！根据前面提到的，我们在回收年轻代的空间时可以把大部分对象都丢掉，少部分活着的对象进入老年代，这样的回收效率是很高的。所以，我们在减少 GC 的开销（只 scan 年轻代，老年代很久才会 scan 一次）的同时也能保证很高的回收效率（大部分对象都死了，所以能够回收很多空间，于是能够work）

​	但是，大数据分析系统中的对象有几个特性。首先，他的生命周期往往很长（**这里面文章是这么讲的，也参考了一些资料，但是我还是要持有怀疑态度，并且通过实验验证这些对象的生命周期真的很长**），然后论文里还说，BDA系统里的对象具有一定的周期性（**我通过一些场景发现并不是所有情况下都是这样，所以我还是要根据文章的参考文献验证 data-intensive 的 task 是否真的具有一定的周期性**）

​	这样的话，我们就发现，当 young generation 满了，回收内存时大部分对象都没有 dead，于是就都去 old generation了。换句话说，我执行了一次 GC，但是我没回收到空间。然后 old generation 满了，我转而去执行 full gc，这样就更加耗时了，所以最后吞吐量很差，而且延迟也很高。

​	然后，作者又发现了一个有意思的事，就是 control path 和 data path（**这部分我还是要持怀疑态度，所以根据他提供的参考文献去看看到底有没有真的如他说的那样：two path，two hypothesis**）control path 跟传统的类 Java 程序很像，data path 的对象生命周期很长，而且回收时具有 **同一周期性**。所以本文就讲 JVM heap 分成两部分，其中 CS 放 control path 的对象，回收时使用 generational GC，而 DS 放 data path的对象，回收时使用 region-based memory management。

​	于是，问题就变成了如下几点：

- 怎么把 heap 给分开呢？
- DS 的 region-based memory management 怎么组织：怎么处理 objects 之间的引用关系，怎么划分region，怎么找escaping objects，怎么决定escaping objects 最终应该去哪？这些操作要高效而又正确；
- 介绍回收 region 的算法以及找 escaping objects 的算法
- 介绍如何回收 CS

## 有哪些闪光点

- Yak 同时提供了 high throughput 和 low latency，这是 Parallel GC 和 CMS 包含的优点：原因是因为用了新的回收算法；
- **data-intensive** system has a clear distinction between a control path and a data path；
- **control path** 主要做 cluster management 、schedule、establish communication channels between nodes、interact with users to parse queries and return results；**data path** 主要做 data manipulation functions that can be connected to form a data processing pipeline（data partitioners, Join or Aggregate，UDF）；
- 我们在做 young GC 的时候，主要是判断一个对象是否 reachable from the **old generation**, a full-heap(major) collection scans both generations 
- 对于 Hadoop，我们在 setup/cleanup API 中加入 *epoch_start* 和 *epoch_end* annotation，这样会不会太大了点；
- The JVM-based implementation enables Yak to work for all JVM-based languages, such as Java, Python, or Scala.
- Tracing GC traces live objects by following references, starting from a set of root objects that are directly reachable from **live stack variables and global variables**. 
- Existing region-based techniques rely heavily on static analyses. An epoch is the execution of a block of data transformation code.
- we have modified the two JIT compilers (C1 and Opto), the interpreter, the object/heap layout, and the Parallel Scavenge collector (to manage the CS）
- A card table groups objects into fixed-sized buckets and tracks which buckets contain objects with pointers that point to the young generation
-  因为 CS 上的 object 是共享的，所以我们可以通过 CS 上的 object 的引用链链接到其他 region 当中；
- **Hyracks runs one JVM on each node with many threads to process data while Hadoop runs multiple JVMs on each node, with each JVM using a small number of threads. 
- Evidence shows that in general the heap size needs to be at least twice as large as the minimum memory size for the GC to perform well. 
- the cluster is 11-node cluster, each with 2 Xeon(R) CPU E5-2640 v3 processor, 32 GB memory, 1 SSD, running CentOS 6.6, the ration between the sizes of the CS and the DS is 1/10. Besides, we ran each program for three iterations, the frist iteration warmed up the JIT.
- We use `pmap` to collect memory consumption periodically.
- Hadoop runs multiple JVMs and different JVM instances are frequently created and destroyed. Since the JVM never returns claimed memory back to the OS until it terminates, the memory consumption always grows for Hyracks and GraphChi.
- Yak was built based on the assumption that in a typical Big Data system, only a small number of objects escape from the data path to the control path. 
- The write barrier and region deallocation are the two major sources of Yak’s application overhead.





## 能够为我们的研究工作提供什么

- Vast opportunities are possible if both user-defined operators and system’s built-in operators are epoch annotated. 
- 首先，Yak 肯定假设 region 里的大部分对象都是存活着的（仅仅是粗粒度的 epoch 标注），所以我们可以利用这点来做文章。每当 region 回收时，我们都会发现有对象肯定会被立刻回收（**目前在Yak里是放入 page list**），而且是大部分对象，所以我们就可以考虑把 memory 的 reallocation 放在region 回收的时候。这点和 ATC 那篇文章很像，目标都在于找到哪些些对象真的不用了，以及能空出多少内存给上层的 framework做调度；
- Yak 给我们提供了很多东西：一套高效并且正确的 reigon-based memory management（**虽然 stack 那一块还是有些缺失，但我还是可以把这个弥补上去**）这个内存方案包含如何组织 DS 的内存回收、CS 和 DS 之间的联动以及如何修改原来的 GC 使整个系统最终work，这些:star:都是值得我们去好好借鉴的，而且是非常有用的！！！结合VLDB那篇文章，我觉得可以思考一下是在 Spark 应用层做这件事还是在 Spark 源码层做这件事；​
- 我们可以自己找些 `data-intensive` 的应用，然后跑一跑实验，调研一下，不要急着做，因为==磨刀不误砍柴工==！！；
- Yak 对于 hadoop 的注解还是很粗粒度的，所以我们从源码层面去**细粒度的注解**是不是真的能够搞出一些名堂出来呢？
- 必要时我们也可以引用一些程序分析的手段用作辅助（这个是很重要的，可以参看 **PLDI 19 Harry Xu** 那篇 Panthera ）；
- 我们可以瞅瞅到底是改Young GC 还是 Full GC，因为不同文章的侧重点都不一样，但是都提出了这个尖锐的问题（可不可以两者一起考虑呢），但是我们可以从 **实验结果** 和 **理论分析** 去决定到底是采用哪种方案；
- 我们在实验时要考虑一个node上几个 JVM，一个 JVM 多少个进程，一个进程多少个线程等等。集群的内存多大，以及 `pmap` 的正确使用；
- ==最后的最后==，我们在确定某一堆对象要一起回收的时候，一定要给出证据，证明真的有很少的 escaping objects（**Yak 中的叫法，而且Yak里面确实给出了相应的百分比，妥妥的实验证据**），或者真的有很少的漏网之鱼，因为这是前提，如果前提都不能满足，那么肯定Yak不能正确工作，自己的代码也不能正确工作。（这可能就是我们之前帮里的combine Yak 不能工作的原因）

[^1]: 见文章第一行