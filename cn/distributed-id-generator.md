我们经常需要给网站里的每个用户赋予一个唯一ID，或者给每条微博一个唯一ID，等等。如何设计一个分布式ID生成器(Distributed ID Generator)？

Follow up: 如何让ID可以粗略的按照时间排序？


### 预估ID的容量

首先，我们需要预估ID的最高容量，举几个例子，

* 短网址。短网址服务需要对互联网上的每个URL，计算出一个短的URL，那么互联网总共有多少个URL呢？ 参考 <http://www.worldwidewebsize.com> 的数据，当前大概有45亿的URL。45亿超过了一个32位无符号整数的范围，用一个64位整数还是绰绰有余的。微博的短网址服务用的是长度为7的字符串，这个字符串可以看做是62进制的数，那么最大能表示$${62}^7=3521614606208$$个网址，刚好在32位整数和64位整数的范围之间。
* 用户ID。这个该怎么预估呢？假设地球上每个人都来注册你的网站，那么你只需要知道地球总人口有多少就行了。当前地球目前总人口约70亿，用一个10位的十进制数来表示绰绰有余了。
* 微博的ID。假设要给每个微博赋予一个唯一的ID，这个ID最大有多大呢？假设我们知道用户每天发送的微博数那就好办了，每天的微博数x356x100，就够100年用了！新浪微博的数据我没查到，不过在这里<http://www.internetlivestats.com/twitter-statistics/>可以查到Twitter的数据，每年用户大概会发 2000亿条tweet, 这个数乘以100，就是 20000000000000，远远小于64位整数，因此可以用一个64位整数存下。

基本上64位整数能够满足绝大多数的场景，我们需要根据具体业务进行分析，缩小范围，毕竟ID短一点，能够节省内存，所以在能够满足百年大计的情况下，越短越好。

下面介绍一些常用的生成ID的方法。


### UUID

用过MongoDB的人会知道，MongoDB会自动给每一条数据赋予一个唯一的[ObjectId](https://docs.mongodb.com/manual/reference/method/ObjectId/),保证不会重复，这是怎么做到的呢？实际上它用的是一种UUID算法，生成的ObjectId占12个字节，由以下几个部分组成，

* 4个字节表示的Unix timestamp,
* 3个字节表示的机器的ID
* 2个字节表示的进程ID
* 3个字节表示的计数器

[UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)是一类算法的统称，具体有不同的实现。UUID的有点是每台机器可以独立产生ID，理论上保证不会重复，所以天然是分布式的，缺点是生成的ID太长，实际中要根据具体场景进行剪裁，生成更短的ID。


### 多台MySQL服务器

既然MySQL可以产生自增ID，那么用多台MySQL服务器，能否组成一个高性能的分布式发号器呢？ 显然可以。

假设用8台MySQL服务器协同工作，第一台MySQL初始值是1，每次自增8，第二台MySQL初始值是2，每次自增8，依次类推。前面用一个 round-robin load balancer 挡着，每来一个请求，由 round-robin balancer 随机地将请求发给8台MySQL中的任意一个，然后返回一个ID。

[Flickr就是这么做的](http://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)，仅仅使用了两台MySQL服务器。可见这个方法虽然简单无脑，但是性能足够好。不过要注意，在MySQL中，不需要把所有ID都存下来，每台机器只需要存一个MAX_ID就可以了。这需要用到MySQL的一个[REPLACE INTO](http://dev.mysql.com/doc/refman/5.0/en/replace.html)特性。

这个方法的一个缺点是，ID是连续的，容易被爬虫抓数据，爬虫基本不用写代码，顺着ID一个一个发请求就是了，太方便了（手动斜眼）。


### Twitter Snowflake

比如 Twitter 有个成熟的开源项目，就是专门生成ID的，[Twitter Snowflake](https://github.com/twitter/snowflake) 。Snowflake的核心算法如下：

![](http://121.40.136.3/wp-content/uploads/2015/04/snowflake-64bit.jpg)

最高位不用，永远为0，其余三组bit占位均可浮动，看具体的业务需求而定。默认情况下41bit的时间戳可以支持该算法使用到2082年，10bit的工作机器id可以支持1023台机器，序列号支持1毫秒产生4095个自增序列id。

[Instagram用了类似的方案](https://engineering.instagram.com/sharding-ids-at-instagram-1cf5a71e5a5c)，41位表示时间戳，13位表示shard Id(一个shard Id对应一台PostgreSQL机器),最低10位表示自增ID，怎么样，跟Snowflake的设计非常类似吧。这个方案用一个PostgreSQL集群代替了Twitter Snowflake 集群，优点是利用了现成的PostgreSQL，容易懂，维护方便。

有的面试官会问，如何让ID可以粗略的按照时间排序？上面的这种格式的ID，含有时间戳，且在高位，恰好满足要求。如果面试官又问，如何保证ID严格有序呢？在分布式这个场景下，是做不到的，要想高性能，只能做到粗略有序，无法保证严格有序。


### 参考资料

* [Sharding & IDs at Instagram](https://engineering.instagram.com/sharding-ids-at-instagram-1cf5a71e5a5c)
* [Ticket Servers: Distributed Unique Primary Keys on the Cheap](http://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
* [Twitter Snowflake](https://github.com/twitter/snowflake)
