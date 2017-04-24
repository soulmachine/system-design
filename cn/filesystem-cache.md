如何设计一个文件系统缓存？

在 Linux 操作系统中，当应用程序需要读取文件中的数据时，操作系统先分配一些内存，将数据从磁盘读入到这些内存中，然后再将数据传给应用程序；当需要往文件中写数据时，操作系统先分配内存接收用户数据，然后再将数据从内存写到磁盘上。文件 Cache 管理指的就是对这些由操作系统分配，并用来存储文件数据的内存的管理。


## 短网址的长度

## 参考资料

* [Linux 内核的文件 Cache 管理机制介绍-IBM](https://www.ibm.com/developerworks/cn/linux/l-cache/)
* [Ticket Servers: Distributed Unique Primary Keys on the Cheap](http://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
* [Twitter Snowflake](https://github.com/twitter/snowflake)
* [短 URL 系统是怎么设计的？](https://www.zhihu.com/question/29270034)
