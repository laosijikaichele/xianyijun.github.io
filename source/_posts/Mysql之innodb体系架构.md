---
title: Mysql之innodb体系架构
date: 2016-05-27 20:52:46
tags: [MySQL] 
---
# **Innodb体系架构** #
![innodb](http://7xrl91.com1.z0.glb.clouddn.com/Diagram1.png)
## **后台线程** ##
Innodb存储引擎是多线程模型，后台有多个不同的后台线程，用来处理不同的任务。
### **Master Thread** ###
Master Thread是个核心的后台线程，它主要是负责将缓冲区池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、undo页的回收等。
### **IO Thread** ###
在Innodb存储引擎中使用AIO来处理写请求。IO Thread主要是负责IO请求的回调处理。IO Thread主要是write、read、insert buffer和IO thread。read thread和write thread数量默认为４，可以通过innodb_read_io_threads和innodb_write_io_threads参数进行设置。
### **Purge Thread** ###
事务被提交后，其使用的undolog可能就不会在需要，因此需要PurgeThread来进行回收已经使用并分配的undo页。
在innodb1.1版本之后，我们可以在mysql数据库配置文件中添加以下配置来启动独立Purge Thread进行处理，来减轻Master Thread的工作，从而提供CPU的使用率和提供性能。
```xml
[mysqld]
innodb_purge_threads = 1
```
在innodb1.2版本之后，可以配置多个purge threads来处理，可以进一步加快undo页的回收。而且Purge Thread需要离散地读取undo页，可以进一步利用磁盘的随机读取性能。
### **Page cleaner Thread** ###
Page cleaner Thread是将原先在Master Thread中将脏页刷新操作放入到独立线程来完成，主要减轻Master Thread的工作和用户查询线程的阻塞，提高存储引擎的性能。
## **内存** ##
![memory](http://7xrl91.com1.z0.glb.clouddn.com/Selection_025.png)
### **缓冲池** ###
Innodb存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。由于CPU速度与磁盘速度之间的差距，通常会使用缓冲区技术来提供数据库性能。
缓冲池是一块内存区域，通过内存来弥补磁盘速度对数据库性能的影响。在数据库进行读取页操作的时候，首先把磁盘读到的页存放在缓冲池中，在下一次读取页操作的时候，首先判断该页是否在缓冲池中，如果该页在缓冲池中，就称该页在缓冲池被命中，直接读取该页，否则就读取在磁盘上的页。
对数据库中页的修改操作，首先会缓冲池中的页，然后根据一定的频率刷新到磁盘中，innodb引擎是通过一种checkpoint的机制将页刷新到磁盘中的。
innodb引擎是可以通过设置innodb_buffer_pool_size来配置缓冲池大小，也可以通过配置innodb_buffer_pool_instances来缓冲池个数，减少数据库内部的资源竞争，增加数据库并发处理的能力。
缓冲池缓存的数据页类型有：索引页、数据页、undo页、插入缓冲(insert buffer)、自适应哈希索引(adaptive hash index)、Innodb存储的锁信息(lock info)、数据字典信息(data dictionary)等。
### **LRU List、Free List和Flush List** ###
缓冲池中缓冲各种类型的页，通过LRU(lastest recent used)最近最少使用算法进行管理，也就是说最频繁使用的页放在LRU列表的首部，最少使用的页放在LRU列表的尾端，如果缓冲池不能存放最新读取到的页，将首先释放的LRU列表中尾端的页。
在innodb引擎中，缓冲池页的大小默认为16KB,同样使用LRU算法对缓冲池进行管理，不过innodb在LRU算法的基础上添加midpoint位置，最新读到的页，并不是直接放到LRU列表的首部，而是放到LRU列表的midpoint位置，在默认配置中，midpoint位置位于LRU列表长度的5/8处,midpoint位置可以通过innodb_old_blocks_pct进行配置。innodb_old_blocks_pct默认值为37，表示新读取的页会插入到LRU列表尾端的37%位置。在innodb存储引擎中，把midpoint之后的列表称为old列表，之前的列表称为new列表。
使用midpoint优化的LRU算法可以防止缓冲池中数据页被刷新出，影响数据库性能。如果新读取的数据直接加到LRU列表的头部，如果读取的数据页比较多(通常为数据或者索引的扫描操作)，可能会把之前活跃的热点数据刷新出去，这样的话，如果下一次需要读取之前活跃热点数据的话，需要重新从磁盘中读取。
我们可以通过设置innodb_old_blocks_time来设置页读取到midpoint位置后需要等待多长时间才会被加入达到LRU列表的new端。
LRU列表是用来管理已经读取的页，数据库刚启动的时候，LRU列表是空的，页都是放在Free List中的。如果需要从缓冲池中分页时，首先从Free列表中查找是否有可用的空闲页，如果有的话就将该页从Free列表中删除，放到LRU List中，否则的话，根据LRU算法，将LRU List中尾端的页淘汰，将该内存空间分配给新的页。当页的old部分加入到new 部分的话，就称此时的操作为page made young,如果由于innodb_old_blocks_time的设置导致页没有从old部分移动到new部分的操作称为page not made young。
我们可以通过show engine innodb status来观察LRU列表的使用情况和运行状态。
Buffer pool size表示当前数据库有多少个页，即8192*16K，总共124M的缓冲池，Free Buffers表示当前Free List中页的数量，Database pages表示当前LRU列表中页的数量。
pages made young表示LRU列表中页移动到前端的次数。
Buffer pool hit rate表示缓冲池的命中率
```
BUFFER POOL AND MEMORY
----------------------
Total memory allocated 137363456; in additional pool allocated 0
Dictionary memory allocated 257292
Buffer pool size   8192
Free buffers       7739
Database pages     453
Old database pages 0
Modified db pages  0
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 453, created 0, written 1
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 453, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
```
Innodb引擎支持压缩页的功能，可以将原来16kb的页压缩成1kb、2kb、4kb、8kb,对于非16kb的页是通过unzip_LRU列表进行管理的,不过LRU列表的页包含了unzip_LRU列表中的页。
unzip_LRU列表中对不同压缩页大小的页分别进行管理，然后通过伙伴算法进行内存分配。例如需要从缓冲池中申请4kb大小的页。

- 首先检查4kb的unzip_LRU列表，检查是否有可用的空闲页。
- 如果有的话，直接分配使用
- 如果没有的话，检查8kb的unzip_LRU列表。
- 如果能够得到空闲页，就将页分为２个4kb页，存放到unzip_LRU列表。
- 如果不能的话，就从LRU列表中申请一个16kb的页，然后把页分为1个8kb的页、2个4kb的页，分配存放到对应的unzip_LRU列表中。

```
LRU len: 453, unzip_LRU len: 0
```
当LRU列表中页被修改后，称该页为脏页，数据库会根据checkpoint机制将脏页刷新回磁盘中，Flush列表中的页即为脏页列表，脏页即存在于LRU列表中，也存在于Flush列表中，LRU列表用来管理缓冲池中页的可用性，Flush列表用来管理将页刷新会磁盘
### **重做日志缓冲** ###
Innodb存储引擎首先将重做日志信息放到重做日志缓冲，然后根据一定的频率将其刷新到重做日志文件，我们可以设置innodb_log_buffer_size设置重做日志缓冲大小，默认为8MB,一般情况下，我们不需要设置太大，通常一秒钟会将重做日志缓冲刷新到日志文件，因此用户只需要保证每秒发生的事务量在这个缓冲大小就可以了。
刷新机制
- Master Thread每一秒将重做日志缓冲刷新到重做日志文件。
- 每个事务提交时会将重做日志缓冲刷新重做日志文件
- 当重做日志缓冲剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件。

### **额外的内存池** ###
在Innodb存储引擎中，对内存的管理是通过内存堆的方式进行的。在对一些数据结构本身的内存进行分配的时候(缓冲池innodb_buffer_pool的帧缓冲frame buffer和对应缓冲控制对象buffer control block)，需要从额外的内存池中进行申请，当该区域的内存不够的时候，会从缓冲池中进行申请。我们可以通过设置参数innodb_additional_mem_pool_size来设置大小
