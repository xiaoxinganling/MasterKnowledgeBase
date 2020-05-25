## Panthera: Holistic Memory Management for Big Data Processing over Hybrid Memories    

> PLDI ’19    

### 1.简要总结

​	非易失性存储（NVM）相较于目前的主存（DRAM）具有容量较大、掉电后仍可存储数据等新特性。考虑到大数据处理任务对内存的需求较大，而现有的内存资源在许多大数据处理任务中常常成为瓶颈，从而使CPU资源被浪费。因此，在已有的DRAM中引入NVM，即混合型存储，能够有效地缓解大数据处理系统的内存压力，同时也能节省内存资源。于是，在引入混合型存储后，作者实现了在此基础上的内存管理方案——Panthera（针对托管型语言，如Java等）。Panthera首先根据静态分析得到程序的一些semantics（哪些对象应该被放入NVM，哪些放入DRAM）,而后修改GC，在底层真正地将每个对象放入到其对应的区域中。Panthera会来一定的额外开销，但能够为大数据处理任务提供较大的内存资源，同时也能够减少能量的消耗（主要是电能）。

​	

### 2. 背景及问题

​	大数据处理任务由于其输入数据规模较大，因此在内存消耗上也尤为显著。已有的研究工作表明，非易失性存储（NVM）能够提供更大的内存空间（相较于DRAM）以及更低的能量损耗（相较于SDD），因此基于混合型存储的系统也备受关注。

​	但是，NVM相较于DRAM具有访问延迟大、带宽小等特点，如果只是简单地将原有的DRAM替换为NVM，应用的性能表现必然会大大降低。当我们使用混合型存储时，必然会存在对象分配和迁移工作。如何在NVM和DRAM之间实现有效的数据存储和迁移工作会影响应用的额外开销（即混合型存储系统带来的开销）。在面临大数据处理系统时，混合型内存的应用还会碰到如下问题：

- JVM 在 OS 的内存管理上增加了一层内存管理方案，由于GC屏蔽了底层细节，并不知道底层是混合型内存，因此NVM和DRAM之间的交互会降低应用的执行性能
- 许多大数据处理系统存在一套内存子系统，即其存在自己的内存管理方案。例如，Spark中使用RDD对分布在各个节点上的数据进行抽象，利用RDD实现内存管理。如何合理地使用混合型内存需要结合RDD等内存管理方式进行思考

​	现有的混合型内存系统并未考虑大数据应用场景。因此作者考虑提出适合大数据处理应用的混合型内存系统。



### 3. 研究思路

​	由于大数据处理应用产生的对象具有一定的周期性，因此它们的生命周期和访问模式也存在一定的特征。例如，Spark中每个RDD包含的所有对象都具有相同的生命周期和访问模式。此外，这些模式再用户程序中很容易被观测出来（通常使用静态分析的手段）。于是，作者提出，不同于在传统应用中，需要分析每个对象的生命周期和访问模式，在大数据处理环境下，我们只需要分析某个集合的生命周期和访问模式即可（集合中每个对象的特征都相同）。

​	上述工作可以用静态分析来完成，但静态分析可能无法捕捉到所有对象集合的特征。因此，作者还使用了动态分析的方案，通过监测对象的访问模式将其放入合适的存储空间中（DRAM或NVM）。

​	在决定哪些RDD应当放入哪些存储空间后，系统底层需要将RDD中的每个对象真正地放入其对应的内存中（NVM或DRAM）。因此，作者还实现了能够利用上述分析结果的GC，对RDD中的对象执行内存分配和迁移。具体内容可总结为如下几部分：

#### 3.1 Static Analysis

​	堆空间分为年轻代和老年代，年轻代由DRAM组成，而老年代由DRAM（小部分）和NVM（大部分）组成。

- 新创建的对象和存储临时或中间数据的short-lived RDD放入年轻代（均为DRAM）中（生命周期短且访问较多）
- 用作缓存的long-lived RDD放入老年代的DRAM中（访问较多）
- 用作容错的long-live RDD放入老年代的NVM中（访问较少）

		静态分析通过为每个RDD打上tag的手段确定其应该存放在哪块内存区域中（如果静态分析不准，还存在运行时分析的手段，这部分在**3.2**中有提及）。思路其实很简单（这些RDD都是被persist（显式）或者调用了action(隐式)）：

- 如果一个RDD在每次循环中被定义，考虑到RDD immutable的属性，它就不会被频繁地使用，因此将其放入NVM中

- 如果一个RDD在循环外被定义，在每次循环中被使用，即被频繁使用，我们将其放入DRAM中

   此外，如果action操作和persist操作没有发生在循环之前或者循环中，我们不会为该RDD打上tag。如果所有的RDD都是NVM的tag，我们将其修改为DRAM以充分利用内存。文中还有对ShuffledRDD的详细处理（*p6*），这里就不再赘述。

#### 3.2 Semantics-aware and physical-memory-aware generational GC 

​	前面的静态分析只是决定RDD应该被放在哪个内存区域中，而真正将每个对象放到相应的内存区域还是需要GC来做。这里的 *semantics-aware* 指的是GC需要利用静态分析得到的tag信息，而 *physical-memory aware*则是指GC需要考虑物理内存（NVM和DRAM）。

- 由于在RDD持久化时会为其分配数组，因此我们将这些对象分配到对应的内存区域中。而要把RDD中所有的对象分配到其对应的内存区域具有一定的困难（这些对象分散在堆空间中，强行将它们找到并移动到对应的内存区域十分耗时，对象的移动可以交给后续的GC来做）
- Minor GC: 扫描年轻代中需要被复制到老年代的DRAM和NVM中的对象（使用tracing 算法去传播tag，这个过程本来就会做，加上一个打tag的操作不会带来多大的开销）
- Major GC: 保证compaction不会发生在DRAM和NVM的边界，并且在Major GC扫描整个堆时根据对象的访问频率（例如，某个RDD被调用了多少次map或reduce方法）重新给其打tag（**3.1**中提到的运行时分析）



### 4. 总结与启发

​	这是一篇结合了硬件的文章，即作者利用新硬件在已有场景下对某些应用进行了优化。作者提出大数据处理应用需要更多的内存，而NVM的出现能够在为其提供足够内存资源的同时，使用更少的能量消耗。与以往优化大数据处理任务的执行时间有所不同，作者实际上给任务带来了额外开销（在可接受范围内），但却大大增加了可用的内存资源。对这篇文章的总结如下：

- Panthera 通过静态分析得到了粗粒度的内存访问模式（每个RDD而不是每个对象），这不仅减少了静态分析的复杂程度，也优化了底层混合型内存的使用。在作者的思路中，经常被访问的对象应当被放在DRAM中，反之则是NVM。**因此，作者才如此执着于access frequency这一特性**。尽管我们的研究工作不会使用NVM，**但其对RDD而不是对象的静态分析为我们的研究提供了新思路**。**如果不需要考虑access frequency这一属性，我们可以考虑跳过用户代码，直接从源码层面分析，这样也不需要额外的静态分析开销**。同时RDD中的对象在生命周期和访问模式上具有相同的特点（这里我们不需要访问模式这个特点），**在源码层面从所有RDD入手不失为一种正确的方案**。

- 上层分析出任务的内存使用特征，下层使用相应的手段根据得到的特征进行真正地操作。作者这样做的原因是JVM层屏蔽了底层内存的细节，不会分辨NVM和DRAM。而我们的工作无需考虑硬件的新特性，如果能结合大数据处理框架层得到的特征直接调用底层JVM提供的相应 API 也能对内存的使用进行优化。**问题就在于，底层是否会提供这些 API，以及执行 API 后系统会不会真的按照我们期望的那样变化。如果可以满足这个条件，我们就不需要对底层已有的机制进行修改，如果不能满足，我们需要根据其所属的情况对其进行小改（增加API）或者大改（提出新的内存管理方案，类似于Yak）**

- 循环的出现使得静态分析的结果更为准确，而循环也正是大数据处理任务内存特征最为明显的一部分，因此，**循环将会是我们优化内存管理方案的重中之重。**

  > The memory tag of an RDD variable is a static approximation of its access pattern, which may not reflect the behavior of all RDD instances represented by the variable at run time. However, user code for data processing often has a simple batch-transformation logic. Hence, the static information inferred from our analysis is often good enough to help the runtime make an accurate placement decision for the RDD  

  此外，以Spark为例，数组也是所有对象中最为频繁出现的一类，**从数组入手取得的效果会比其他对象更好**。（machine learning, graph analytics, stream processing等应用中数组也被广泛应用）   

- 文章对Spark基础和RDD特性的介绍比较全面，可供日后翻阅（p3-p5）

  

### 5. 论文参考

- Martin Maas, Krste Asanović, Tim Harris, and John Kubiatowicz. 2016. Taurus: A Holistic Language Runtime System for Coordinating Distributed Managed-Language Applications. In Proceedings of the TwentyFirst International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS ’16).     
- Mingyu Wu, Ziming Zhao, Haoyu Li, Heting Li, Haibo Chen, binyu Zang, and Haibing Guan. 2018. Espresso: Brewing Java For More Non-Volatility. In Proceedings of the Twentieth International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS ’18)    