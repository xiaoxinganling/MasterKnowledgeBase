# An Experimental Evaluation of Garbage Collectors on Big Data Applications

## 研究背景

​	本文的研究背景是使用 GC 的大数据分析系统，本文旨在挖掘在大数据分析系统中使用 GC 遭遇的问题及其**具体**原因。考虑到很多文章都发现了 GC 会拖慢大数据分析任务的执行，他们也根据所观测的现象及原因给出了相应的解决方案。这些解决方案有如下缺点：

- *缺乏深度*：大部分研究者只根据实验结果得出了“表面”的结论。例如，通过分析程序运行时间得出 GC 会占据应用程序总时间的 50% 以上，但并未深入探讨 GC 的何种特性导致其如此耗时。再者，部分研究者观察出大数据分析任务的周期性会导致暂停时间很长的 GC（Yak），但其并未深入探讨周期性的来源，而是把这一工作交给程序员去完成（即通过手动添加注解达到优化程序执行的目的）。因此，对大数据分析系统和 GC 更加深入的理解能够帮助我们更好地解决 GC 在大数据分析任务中面临的困境。
- *缺乏广度*：经过粗略的统计，我们发现 GC 在大数据分析任务中表现不好的原因大致有如下几点。首先，数据规模或者数据包含的信息量很大（例如一些很大的图）；其次，GC 的设计初衷不是面向大数据分析系统，因此在很多时候并不适用于处理大数据分析任务；再次，许多大数据分析任务都只包含 *data-level* 而不是 *object-level* 的内存管理，即根据应用程序在运行过程中产生数据的种类进行内存管理（如，Spark Job将一些缓存的数据存入 JVM 堆的 Storage Space；将执行过程中产生的中间数据存入 JVM 堆的 Execution Space）。许多研究者都只考虑上述因素的某一点，为了更加深入地理解大数据分析系统与 GC 的联系，我们需要综合上述因素给出**具有说服力**的结论和可能的解决方案。

## 问题难度

​	许多人都能说出 GC 不适用于大数据分析系统，但他们无法说出确切的原因。或许他们能说出 GC 的 *pattern* 与大数据分析任务的 *pattern* 不匹配，因此 GC 在大数据分析系统中表现得很差，但他们却不能说出 GC 和大数据分析系统的 *pattern* 究竟是什么。本文解决的就是这样一个问题：到底大数据分析系统的 *pattern* 是什么，究竟 GC 的特征又是什么，实验表明 GC 带来了很大的性能降级，这究竟是为什么？

​	如果能阐述清楚上面几个问题，我们就能”对症下药“，针对特定问题给出具体的解决方案。这个过程分两部分，首先，要阐述清楚原因，其次，要针对现有的「GC 在大数据分析系统中表现很差」的问题给出一定的解决方案。难点在于对 GC 和大数据分析系统 *pattern* 的**总结**，即，我们能根据某一次实验，某一张图给出某个确切问题的概括，但如果要我们指出某个大的系统的特点，问题突然变得有些无从下手。

## 本文动机

​	本文就是要做这样一件其他人没有做过的事：**总结** GC 和大数据分析系统的 *pattern*，为后续解决该类问题的研究者指出大致的方向。

## 创新点和贡献

​	为了对大数据分析系统和 GC 进行概括，本文选取了 4 种 Spark 大数据分析任务（GroupBy [Spark SQL], Join [Spark SQL], SVM [Machine Learning], PageRank [Graph Computing]）,3 种老年代 GC[^1]（Parallel，CMS，G1）进行实验。根据实验结果，本文得到了几个结论及启发点，这些启发点从不同角度分别为程序员和研究者提供了不同的建议。

## 算法、实现、验证

### 大数据分析系统特征

#### Memory pattern

- *long-lived accumulated records* or *iterative long-lived accumulated records*

  在 shuffle read 的过程中，shuffle fetch 的数据会存储在内存中一直到 reduce 阶段结束（***long-lived***）。这些数据以 HashMap的形式存储在内存中，其中某一条记录会根据其对应的 key 累积地存入 HashMap 相应的 value中（***accumulated***）。在 PageRank 中，迭代计算会产生许多 iteration，每个 iteration 都会产生 *long-lived accumulated records*，并且上一个 iteration 产生的 records 在当前 iteration 中不会被用到（***iterative***）。

- *cached records*

  许多输入数据都会被缓存在内存中，直到 task 执行结束（***cached***）

- massive temporary records

  reduce 阶段结束前会产生大量临时数据，这些数据占用的内存在任务结果写入磁盘或者发送给 driver 后会被回收（***massive、temporary***）

- humongous data objects

  机器学习任务（SVM）会产生许多大对象，例如，SVM 中的每个参数数组有几百MB（**humongous**）

#### Computation feature

- iterative computation：许多机器学习任务和图计算任务具有迭代性/周期性
- CPU-intensive data operators：用户定义的操作在通常情况下对 CPU 的需求都很大

### GC 特征

- 三种老年代 GC 在处理 ***long-lived shuffled data objects*** 和 ***humongous data objects*** 时会出现多次长时间的暂停（问题的关键在此）
- 并发式 GC [^2]在大数据分析系统中较 STW（stop-the-world）GC[^3] 表现更好，但其对 CPU 资源的需求更大

### 结论及启发

### 对程序员的建议

### 对研究者的建议

​	这三点在组会 ppt 的 **Finds and Implications** 和 **Lessons and Insights** 有详细介绍，此处不再赘述。

## 读后感

- 本文提到的 [*SparkProfiler*](https://github.com/JerryLead/SparkProfiler) 为我们后续的 dynamic memory management 提供了实验基础
-  我们可以将重点放在本文提到的 ***long-lived accumulated objects*** 、 ***cached objects*** 以及 ***humongous objects***上，这些对象具有鲜明的特征，我们可以找出这些对象在何时不会被用到，该信息可以用数据分析框架层（Spark or MonoSpark）告诉我们
- 在实现 region-based memory management 的过程中，我们需要考虑内存利用率和程序稳定性之间的平衡[^4]
- CMS 和 Parallel 分别使用流处理和批处理任务，我们可以针对任务种类选择不同的 GC 并对其进行优化

[^1]: 因为老年代 GC 占据了 GC 的大部分时间，因此本文选择老年代 GC 进行研究
[^2]: 并发式 GC（concurrent GC）：GC 线程在工作时不暂停用户线程，即与用户线程同时工作。其中，CMS 在 mark 和 sweep 阶段都是并发的，而 G1 只在 mark 阶段进行并发操作
[^3]: STW GC（Stop-The-World GC）：GC 线程在工作时会暂停所有用户线程
[^4]: region-based memory management 在处理大对象时可能会面临单个 region “放不下”的情况，这时候我们可以适当增加每个 region 的大小，但这样会导致内存利用率的降低。因此，内存利用率和程序稳定性（不产生 OOM 错误）之间存在 trade-off