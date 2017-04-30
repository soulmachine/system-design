请设计一个线程安全的HashMap。

先回顾一下普通的哈希表(HashMap)是怎么写出来的，再讨论如何做到线程安全。HashMap的核心在于如何解决哈希冲突，主流思路有两种，

* 开地址法(Open addressing). 即所有元素在一个一维数组里，遇到冲突则按照一定规则向后跳，假设元素x原本在位置`hash(x)%m`（m表示数组长度），那么第i次冲突时位置变为`Hi = [hash(x) + di] % m`，其中`di`表示第i次冲突时人为加上去的偏移量。偏移量`di`一般有如下3种计算方法，

    1. 线性探测法(Linear Probing)。非常简单，发现位子已经被占了，则向后移动1位，即$$d_i = i$$, i=1,2,3,...

        该算法最大的优点在于计算速度快，对CPU高速缓存友好；其缺点也非常明显，假设3个元素x1，x2，x3的哈希值都相同，记为p, x1先来，查看位置p，是空，则x1被映射到位置p，x2后到达，查看位置p，发生第一次冲突，向后探测一下，即p+1，该位置为空，于是x2映射到p+1, 同理，计算x3的位置需要探测位置p, p+1, p+2，也就是说对于发生冲突的所有元素，在探测过程中会扎堆，导致效率低下，这种现象被称为clustering。

    1. 二次探测法(Quadratic Probing)。$$d_i=ai^2+bi+c$$, 其中a,b,c为系数，$$d_i$$是i的二次函数，所以称为二次探测法。该算法的性能介于线性探测和双哈希之间。
    1. 双哈希法(Double Hashing)。偏移量di由另一个哈希函数计算而来，设为`hash2(x)`，则`di=(hash2(x) % (m-1) + 1) * i`

* 拉链法(Chaining)。开一个定长数组，每个格子指向一个桶(Bucket, 可以用单链表或双向链表表示)，对每个元素计算出哈希并取模，找到对应的桶，并插入该桶。发生冲突的元素会处于同一个桶中。

JDK7和JDK8里`java.util.HashMap`采取了拉链法。

如何将基于拉链法的HashMap改造为线程安全的呢？有以下几个思路，

* 将所有public方法都加上synchronized. 相当于设置了一把全局锁，所有操作都需要先获取锁，`java.util.HashTable`就是这么做的，性能很低
* 由于每个桶在逻辑上是相互独立的，将每个桶都加一把锁，如果两个线程各自访问不同的桶，就不需要争抢同一把锁了。这个方案的并发性比单个全局锁的性能要好，不过锁的个数太多，也有很大的开销。
* 锁分离(Lock Stripping)技术。第2个方法把锁的压力分散到了多个桶，理论上是可行的的，但是假设有1万个桶，就要新建1万个`ReentrantLock`实例，开销很大。可以将所有的桶均匀的划分为16个部分，每一部分成为一个段(Segment)，每个段上有一把锁，这样锁的数量就降低到了16个。JDK 7里的`java.util.concurrent.ConcurrentHashMap`就是这个思路。
* 在JDK8里，ConcurrentHashMap的实现又有了很大变化，它在锁分离的基础上，大量利用了了CAS指令。并且底层存储有一个小优化，当链表长度太长（默认超过8）时，链表就转换为红黑树。链表太长时，增删查改的效率比较低，改为红黑树可以提高性能。JDK8里的ConcurrentHashMap有6000多行代码，JDK7才1500多行。


### 方案1: JDK7版本，锁分离

TODO


### 方案2: JDK8版本，锁分离+CAS+红黑树

TODO


### 参考资料

* [ConcurrentHashMap.java - JDK8](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/util/concurrent/ConcurrentHashMap.java)
* [ConcurrentHashMap.java - JDK7](http://hg.openjdk.java.net/jdk7/jdk7/jdk/file/tip/src/share/classes/java/util/concurrent/ConcurrentHashMap.java)
* [Java 8系列之重新认识HashMap - 美团点评技术团队](http://tech.meituan.com/java-hashmap.html)
* [JDK1.8源码分析之HashMap（一） - 博客园](http://www.cnblogs.com/leesf456/p/5242233.html)
* [ConcurrentHashMap - 博客园](http://www.cnblogs.com/yydcdut/p/3959815.html)
* [探索jdk8之ConcurrentHashMap 的实现机制 - 博客园](http://www.cnblogs.com/huaizuo/p/5413069.html)
* [《Java源码分析》：ConcurrentHashMap JDK1.8 - CSDN](http://blog.csdn.net/u010412719/article/details/52145145)
* [ConcurrentHashMap源码解读(put/transfer/get)-jdk8](https://bentang.me/tech/2016/12/01/jdk8-concurrenthashmap-1/)
* [ConcurrentHashMap源码分析（JDK8版本）](http://blog.csdn.net/u010723709/article/details/48007881)
* [ConcurrentHashMap源码分析--Java8](http://note.youdao.com/share/?id=dde7a10b98aee57676408bc475ab0680&type=note/)
* [聊聊并发（四）——深入分析ConcurrentHashMap - InfoQ](http://www.infoq.com/cn/articles/ConcurrentHashMap)
* [探索 ConcurrentHashMap 高并发性的实现机制 - IBM](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/)
* [ConcurrentHashMap源码解析 - Github](https://github.com/BingLau7/blog/issues/14)
* [并发与并行的区别？ - 知乎](https://www.zhihu.com/question/33515481/answer/105348019)
* [深入理解Java内存模型（一） - 基础](http://www.infoq.com/cn/articles/java-memory-model-1)
* [深入理解Java内存模型（二） - 重排序](http://www.infoq.com/cn/articles/java-memory-model-2)
* [深入理解Java内存模型（三） - 顺序一致性](http://www.infoq.com/cn/articles/java-memory-model-3)
* [深入理解Java内存模型（四） - volatile](http://www.infoq.com/cn/articles/java-memory-model-4)
* [深入理解Java内存模型（五） - 锁](http://www.infoq.com/cn/articles/java-memory-model-5)
* [深入理解Java内存模型（六） - final](http://www.infoq.com/cn/articles/java-memory-model-6)
* [深入理解Java内存模型（七） - 总结](http://www.infoq.com/cn/articles/java-memory-model-7)
