# NG2C

## 简要阐述干了什么

​	由于GC的存在，大数据处理应用面临着不可预测的长时间暂停等问题。问题的关键在于GC假定大多数对象“朝生夕灭”，而大数据处理应用会产生大量生命周期较长的中间对象，作者认为大量的对象复制（object copy）是导致GC暂停的主要原因，因此，本文提出了一种新型GC，它支持用户自定义**动态**的JVM 代（generation），并将具有类似生命周期特征（lifetime profile）的对象存放在同一代中，这样就可以减少对象复制操作，从而大大减少应用暂停时间。

## 背景

​	这里的背景其实和[Yak](./conclusion_for_yak.md)提到的背景类似，但作者将重点集中在GC中对象晋升（object promotion）和内存压缩（heap compaction）导致的对象复制（objecte copy）上。大量的in-memory操作（如缓存的使用等）会产生许多中间对象，而这些对象的生命周期很长，这与GC的假设并不相符。因此GC在回收他们时会触发大量对象复制操作，从而导致暂停时间大大增加。因此，作者希望通过减少对象复制操作在保证吞吐量的情况下减少GC暂停时间。前文提及，对象复制主要由promotion和heap compaction导致，前者是年轻代中经过几次young GC后仍然存活的对象会被复制到老年代中，后者是老年代中生命周期不同的对象在被回收时会产生内存碎片，而在整合碎片时会出现对象复制操作。为了优化上述两者情况，作者提出支持用户自定义动态的generation，并将具有类似生命周期的对象放在同一generation中，这样可同时减少promotion和heap compaction引起的对象复制操作，从而降低应用的长时间暂停。

​	该方案的关键在于如何实现JVM heap的**multiple** and **dynamic** generation（multiple体现在generation不止两个，可由用户指定，而dynamic则指generation在运行时可被动态创建和销毁，并且大小可变）、如何确定对象的生命周期（object lifetime profiler，即确定对象应当被分配到哪个generation）以及如何实现正确而高效的内存分配与回收等。

## 解决思路

### heap layout

​	堆空间除了原来的young generation和old generation外，还包含众多dynamic generation，他们用于存放生命周期类似的对象，并且generation的大小就是其包含的所有对象的大小。为了避免object promotion，新创建的对象可以被分配到任一dynamic generation或者old generation。

### memory allocation/collection

​	如果对象在创建时使用了@Gen注解，则会被分配到当前的dynamic generation中，否则被分配在young generation中；而内存回收主要包含三种情况：

- minor collection：和young GC一致
- mixed collection：回收年轻代和存活对象较少的dynamic generation以及老年代，存活对象被复制到老年代中
- full connection：回收所有generation，存活对象被复制到老年代中

（:star:为什么增加了一种mixed collection呢？意义何在？）

### object lifetime recorder

​	与POLM2类似，NG2C使用Object Lifetime Recorder profiler得到对象的生命周期，并分析出对象应当被分配到哪个generation中。核心思想：根据对象分配情况+堆快照（GC执行信息）=> object graph（决定对象分配到哪个generation中）。由此可见，这是一种离线方法，需要在程序正式运行前执行一次Object Lifetime Recorder profiler，再根据profiler为用户代码添加相应注解，添加完毕后才能执行。

## 思考与启发

​	总的来说，这篇文章将优化大数据处理系统的内存使用集中至一点，即减少对象的复制。全文的所有思路以及具体实现都是为这一思想而服务的，为了减少promotion和compaction带来的对象复制，作者将对象按照其生命周期分配到不同的generation中。而本文的重点也在于如何获取正确的对象生命周期。作者采用的是对作业进行profiler，尽管准确，但却是一种离线的方法。当作业的规模变大时，离线执行并且在用户代码中添加注解变得异常麻烦，这大大降低了其实用性。通过阅读这篇文章，我的启发如下：

- 可从某个点切入，从而解決具体的问题，因此我打算将点放在迭代式应用、shuffle-intensive应用的内存优化上，旨在使用较少的代价分析出这些应用的对象特性；
- 作者后期希望将object lifetime recorder整合到JVM中以实现在线的操作，但这一过程过于复杂，且特殊性太强，本想继续follow，但经考虑后暂时放弃；
- 注解的添加如果是在framework中，或其他对用户透明的地方，系统的实用性会更上一层，因此我们也可考虑在framework层添加注解**直接或间接**得到统计信息用于指导内存优化。