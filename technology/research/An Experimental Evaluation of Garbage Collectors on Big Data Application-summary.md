## 几句话说清楚文章干的事

​	由于 Java 和 Scala 等基于自动内存管理方案的语言对编程人员十分友好，因此现有的大数据处理框架，如 Hadoop MapReduce，Spark 等都基于这类语言编写，大数据处理任务的内存回收也依赖其自动内存管理方案 ，即垃圾回收机制（GC）。由于垃圾回收机制的设计初衷并非用于处理大规模数据集，因此我们在一些实验中发现大数据处理任务对 GC 的效率十分敏感。换句话说，GC 的存在会显著影响这些大数据处理任务的性能。

​	我们不能将性能变差简单地归结于数据规模的变大。因此，这篇文章分析了 Spark 中 4 类典型应用的计算特征和内存模式以及 3 种常用 GC 的回收策略，并在使用 3 种不同 GC 的环境下对上述四类应用都进行了实验，从而更加深入地理解了 GC 对大数据处理任务产生的影响。最后，作者基于上述发现对开发者和研究者都提出了相应的优化建议。

## 文章主体内容

### 背景介绍

​	通过对这篇文章之前的文献做的研究，我们可以将 GC 在大数据处理任务中表现不佳的原因简单概括为如下几点：

- 任务执行过程中伴随着**大量**输入数据以及中间结果，这些数量庞大的对象给 GC 带来了显著的压力；
- 现有的大数据处理框架都**只**支持`data-level` 的内存管理（与 JVM 中的 GC 等 `object-level` 的内存管理相对应）。例如，在 Spark 中，内存在逻辑上被分为两部分，一部分用于 `shuffling & data caching`，另一部分用于存储在任务执行过程中产生的中间结果。当前者空间不足时，后者无法将空闲的空间分配给前者；
- 现有的 `object-level` 的内存管理方案，即 GC，在分配和回收内存时并没有考虑大数据处理任务中的对象特性，这导致 GC 经常会拖慢任务的执行时间甚至造成任务执行失败。

### 实验分析

​	在此基础上，本文想结合*对象特征*、*大数据处理框架的内存管理*  和 *GC* 这三个角度彻底理解 GC 对大数据处理任务产生的影响。通过分析实验结果，本文为我们提供了如下关键点：

- 大数据处理任务的 `memory usage patterns` 和 `computation features` 是导致其在不同 GC 的环境下出现性能差异的主要原因，也是其性能会受到 GC 显著影响的原因。

  其中 `memory usage pattern` 主要包括以下两点：

  - long-lived shuffled data：在 shuffle 阶段，大量的对象会“累积”在内存中，并且这部分对象的生命周期很长；
  - humongous data objects：某些大数据处理任务（例如，某些机器学习任务，如 SVM 等）会产生单个 “很大” 的对象（例如，长度很大的数组等）。

  `computation features` 包括以下两点：

  - iterative computation：某些大数据处理任务（例如某些图计算任务，如 PageRank）会迭代很多个周期，在每个周期里可能会产生 long-lived shuffled data；
  - CPU-intensive data operators：大部分大数据处理任务包含 CPU-intensive 的 operators（如 join）。

- 相较于 stop-the-world 的GC（例如 Parallel），并发的 GC（例如 CMS 和 G1）的部分阶段会与大数据处理任务进程并发执行，这会显著减少 GC 的暂停时间，但也会导致 GC 与 CPU-intensive 的应用竞争 CPU；

- 现有的 GC 在处理 `humongous data objects`（巨大对象）时表现都很不好，尤其是使用非连续 `region` 的 G1 Collector，因为这类对象需要大量连续的空间。

### 给程序员和研究者的建议

​	通过对上述发现的详细分析，作者给应用程序开发者和研究者提出了几项建议。

​	对于开发者，建议主要有以下三点：

- 尽量减少应用程序中 `long-lived accumulated objects`（之前提到在shuffle阶段会累积在内存中的对象）的出现 —— 将 Spark 中做 data shuffle 的数据结构从 HashTable 替换为更有效的 [*Compressed Buffer Tree*](http://delivery.acm.org/10.1145/2530000/2523625/a18-amur.pdf?ip=114.212.82.69&id=2523625&acc=OA&key=BF85BBA5741FDC6E.180A41DAF8736F97.4D4702B0C3E38B35.FB8CB00F758AD1E5&__acm__=1572353607_1453e6d0a07cc7066594008613c83338) 或者在使用自定义的 `operator` 时减少其空间复杂度；
- 尽量不使用 `humongous data objects` （巨大对象）—— 将其分为许多小对象 or 如果使用 G1 Collector，则相应增加 G1 Collector 的 region size；
- 对于会产生 `long-lived accumulated objects` 的任务，我们尽量选择 CMS 或者 G1 Collector。相较于 Parallel Collector，这两者的 GC 暂停时间更短。在使用 CMS 或者 G1 Collector 时，我们要注意为每个 task 多分配一些 CPU core，因为前面提到这两个 GC 会和 CPU-intensive 的 operator 竞争 CPU。

​	对于研究者，建议主要有以下几点：

- **Reduce the GC frequency via *prediction-based heap sizing policy*** ：由于之前在实验中观察到，JVM 的 young/old generation 在数据处理任务的执行过程中发生了不合理的变化（两者之间的界限发生变化，内存总量并未发生改变），从而导致 Full GC 暂停时间增加。因此，我们可以根据预测的 memory usage 动态地调整 young/old generation 之间的界限。因为 Spark 知道 shuffled records 的总大小，我们可以在 Spark framework 这一层采用线性回归预测当前 JVM 的 memory usage，从而指导 young/old generation 大小的动态调整；（注意：两者之和，即总内存大小并未发生改变）
- **Minimize the GC work via *lifecycle-aware* object marking algorithm**：目前的 GC 在判断对象是否死亡时几乎需要遍历所有的对象，这十分耗时。但在具有 `long-lived accumulated objects` 的数据处理任务中，Spark framework 可以告诉 GC 哪些对象不再使用（例如，shuffled records spill 到磁盘上或者 cached data 不需要再 cache等），这样 GC 不用遍历这些对象并判断它们是否死亡就可以直接将其回收；
- **Minimize GC work in iterative applications via *overriding-based* object sweeping algorithm** ：对于那些具有 `iterative long-lived accumulated objects` 的数据处理任务，我们可以使用一个 region 存放一次 iteration 产生的所有对象。当下一次 iteration 开始时，上个 iteration 产生的对象可以被回收，我们直接在 region 中**覆盖写入**这次 iteration 产生的对象，无需花费额外的时间将其清除掉。

## 给我们的研究工作带来的思考

- 之前金熠波师兄和我交流时提及我搭建的集群内存太少，其实我可以从这些文章中找到他们做实验时集群的配置如何，进而配置相应的实验环境。除此之外，本次实验使用了作者自己实现的 [*SparkProfiler*](https://github.com/JerryLead/SparkProfiler)，该工具对我们后续实验部分的参考价值巨大；
- 尽管这篇文章**大部分**都在介绍三种 GC（Parallel，CMS，G1）的表现如何，但我还是从中吸取了一些很有价值的信息：
  - 三种 GC 在面对大数据处理任务表现有所侧重，这让我对三者的取舍有一个大体的把握；
  - 本文在讨论 memory pattern 时提到的几种具有显著特点的对象（*accumulated records*，*temporary output records*，*humongous data objects*，*iterative accumulated records* 以及 *cached records*）都在 MonoSpark 中有所映射，在此基础上我可以更好地挖掘 *monotask* 的对象特征。除此之外，我发现 Spark 并不是在所有情况下都适用于 Yak（只有 *iterative accumulated records* 适合 Yak），这与我上次和 Harry Xu交流时问他为何在实验时选择 Hadoop而不是 Spark 时得到的回答一致。
  - 本文对开发者给出的三个建议有两个建议（第二个和第三个）都对 **弹性内存分配** 具有很大的指导作用，因此下一步我打算进一步细化 MonoSpark 在提供指导作用时实际具备的功能（在和Spark比较时多出的指导作用）以及再找一些相关文章阅读。