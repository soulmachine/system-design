寻找数据流中出现最频繁的k个元素(find top k frequent items in a data stream)。这个问题也称为 Heavy Hitters.

这题也是从实践中提炼而来的，例如搜索引擎的热搜榜，找出访问网站次数最多的前10个IP地址，等等。


### 方案1: HashMap + Heap

用一个 `HashMap<String, Long>`，存放所有元素出现的次数，用一个小根堆，容量为k，存放目前出现过的最频繁的k个元素，

1. 每次从数据流来一个元素，如果在HashMap里已存在，则把对应的计数器增1，如果不存在，则插入，计数器初始化为1
1. 在堆里查找该元素，如果找到，把堆里的计数器也增1，并调整堆；如果没有找到，把这个元素的次数跟堆顶元素比较，如果大于堆丁元素的出现次数，则把堆丁元素替换为该元素，并调整堆

* 空间复杂度`O(n)`。HashMap需要存放下所有元素，需要`O(n)`的空间，堆需要存放k个元素，需要`O(k)`的空间，跟`O(n)`相比可以忽略不急，总的时间复杂度是`O(n)`
* 时间复杂度`O(nlogk)`。每次来一个新元素，需要在HashMap里查找一下，需要`O(1)`的时间；然后要在堆里查找一下，`O(logk)`的时间，有可能需要调堆，又需要`O(logk)`的时间，总的时间复杂度是`O(nlogk)`。

如果元素数量巨大，单机内存存不下，怎么办？ 有两个办法，见方案2和3。


### 方案2: 多机HashMap + Heap

* 可以把数据进行分片。假设有8台机器，第1台机器只处理`hash(elem)%8==0`的元素，第2台机器只处理`hash(elem)%8==1`的元素，以此类推。
* 每台机器都有一个HashMap和一个 Heap, 各自独立计算出 top k 的元素
* 把每台机器的Heap，通过网络汇总到一台机器上，将多个Heap合并成一个Heap，就可以计算出总的 top k 个元素了


### 方案3: Count-Min Sketch + Heap

既然方案1中的HashMap太大，内存装不小，那么可以用[Count-Min Sketch算法](frequency-estimation.md)代替HashMap，

* 在数据流不断流入的过程中，维护一个标准的Count-Min Sketch 二维数组
* 维护一个小根堆，容量为k
* 每次来一个新元素，
    * 将相应的sketch增1
    * 在堆中查找该元素，如果找到，把堆里的计数器也增1，并调整堆；如果没有找到，把这个元素的sketch作为钙元素的频率的近似值，跟堆顶元素比较，如果大于堆丁元素的频率，则把堆丁元素替换为该元素，并调整堆

这个方法的时间复杂度和空间复杂度如下：

* 空间复杂度`O(dm)`。m是二维数组的列数，d是二维数组的行数，堆需要`O(k)`的空间，不过k通常很小，堆的空间可以忽略不计
* 时间复杂度`O(nlogk)`。每次来一个新元素，需要在二维数组里查找一下，需要`O(1)`的时间；然后要在堆里查找一下，`O(logk)`的时间，有可能需要调堆，又需要`O(logk)`的时间，总的时间复杂度是`O(nlogk)`。


### 方案4: Lossy Counting

Lossy Couting 算法流程：

1. 建立一个HashMap<String, Long>，用于存放每个元素的出现次数
1. 建立一个窗口（窗口的大小由错误率决定，后面具体讨论）
1. 等待数据流不断流进这个窗口，直到窗口满了，开始统计每个元素出现的频率，统计结束后，每个元素的频率减1，然后将出现次数为0的元素从HashMap中删除
1. 返回第2步，不断循环

Lossy Counting 背后朴素的思想是，出现频率高的元素，不太可能减一后变成0，如果某个元素在某个窗口内降到了0，说明它不太可能是高频元素，可以不再跟踪它的计数器了。随着处理的窗口越来越多，HashMap也会不断增长，同时HashMap里的低频元素会被清理出去，这样内存占用会保持在一个很低的水平。

很显然，Lossy Counting 算法是个近似算法，但它的错误率是可以在数学上证明它的边界的。假设要求错误率不大于ε，那么窗口大小为1/ε，对于长度为N的流，有N／（1/ε）＝εN 个窗口，由于每个窗口结束时减一了，那么频率最多被少计数了窗口个数εN。

该算法只需要一遍扫描，所以时间复杂度是`O(n)`。

该算法的内存占用，主要在于那个HashMap, Gurmeet Singh Manku 在他的论文里，证明了HashMap里最多有 `1/ε log (εN)`个元素，所以空间复杂度是`O(1/ε log (εN))`。


### 方案5: SpaceSaving

TODO, 原始论文 "Efficient Computation of Frequent and Top-k Elements in Data Streams"


### 参考资料

1. [An improved data stream summary:the count-min sketch and its applications](http://vaffanculo.twiki.di.uniroma1.it/pub/Ing_algo/WebHome/p14_Cormode_JAl_05.pdf) by Graham Cormode
1. [Approximate Frequency Counts over Data Streams](http://delab.csd.auth.gr/courses/c_dm_pms/afc.pdf) by Gurmeet Singh Manku
1. A.Metwally, D.Agrawal, A.El Abbadi. Efficient Computation of Frequent and Top-k Elements in Data Streams. In Proceeding of the 10th International Conference on Database Theory(ICDT), pp 398-412,2005.
1. Massimo Cafaro, et al. “A parallel space saving algorithm for frequent items and the Hurwitz zeta distribution”. Proceeding arXiv: 1401.0702v12 [cs.DS] 19 Setp 2015.
1. [Efficient Computation of Frequent and Top-k Elements in Data Streams](http://www.cse.ust.hk/~raywong/comp5331/References/EfficientComputationOfFrequentAndTop-kElementsInDataStreams.pdf) by Ahmed Metwally
1. [Finding Frequent Items in Data Streams ](http://dmac.rutgers.edu/Workshops/WGUnifyingTheory/Slides/cormode.pdf)
* [实时大数据流上的频率统计：Lossy Counting Algorithm - 待字闺中](http://www.wdiandi.com/p/b3779f.shtml)
* [What is Lossy Counting? - Stack Overflow](http://stackoverflow.com/a/8033083/381712)
