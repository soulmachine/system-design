如何设计一个文件系统缓存？

在 Linux 中，当一个程序需要读取磁盘上的文件时，Linux先分配一块内存，将数据从磁盘读入到这块内存中，然后再将数据传给这个程序。当需要往文件中写数据时，Linux先分配内存接收用户数据，然后再将数据从内存写到磁盘上。如何设计这样一个文件系统缓存机制？


## 短网址的长度

## 参考资料

* [Linux 内核的文件 Cache 管理机制介绍-IBM](https://www.ibm.com/developerworks/cn/linux/l-cache/)
* [Ticket Servers: Distributed Unique Primary Keys on the Cheap](http://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
* [Twitter Snowflake](https://github.com/twitter/snowflake)
* [短 URL 系统是怎么设计的？](https://www.zhihu.com/question/29270034)
