# Redis

## redis持久化

Redis 的持久化主要有两大机制，即 AOF（Append Only File）日志和 RDB 快照。

### AOF日志

AOF日志是写后日志。redis是先执行命令，把数据写入内存，然后才记录日志。

![img](https://gitee.com/wanghengg/picture/raw/master/2020/407f2686083afc37351cfd9107319a1f.jpg)

**写后日志的好处：**AOF里面记录的是redis收到的每一条命令，写后日志可以避免检查命令是否出错，如果命令错误，客户端会直接报错，只有正确的命令才能被写入AOF日志。

**AOF日志的风险**：如果刚执行完一个命令，还没来得及记日志就宕机了，那么相应的命令和数据就有丢失的风险。

**AOD配置项**

* **Always**，同步写回：每个写命令执行完，立马同步地将日志写回到磁盘

* **Everysec**，每秒写回：每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，每隔一秒把缓冲区的内容写入到磁盘。

* **No**，操作系统控制的写回：每个命令执行完，只是先把日志写到AOF文件的内存缓冲区，由操作系统决定何时将缓冲区内容写回到磁盘。

* ![img](https://gitee.com/wanghengg/picture/raw/master/2020/72f547f18dbac788c7d11yy167d7ebf8.jpg)

  **日志文件太大怎么办？  <font color=red>重写机制</font>**

  AOF 重写机制就是在重写时，Redis 根据数据库的现状创建一个新的 AOF 文件，也就是说，读取数据库中的所有键值对，然后对每一个键值对用一条命令记录它的写入。比如说，当读取了键值对“testkey”: “testvalue”之后，重写机制会记录 set testkey testvalue 这条命令。这样，当需要恢复时，可以重新执行该命令，实现“testkey”: “testvalue”的写入。

  和 AOF 日志由主线程写回不同，重写过程是由后台子进程 bgrewriteaof 来完成的，这也是为了避免阻塞主线程，导致数据库性能下降。**重写日志由另一个进程实现，不会阻塞主线程，主进程会fork出一个子进程专门用于重写日志**

  ![img](https://gitee.com/wanghengg/picture/raw/master/2020/6b054eb1aed0734bd81ddab9a31d0be8.jpg)

### 内存快照

和 AOF 相比，RDB 记录的是某一时刻的数据，并不是操作，所以，在做数据恢复时，我们可以直接把 RDB 文件读入内存，很快地完成恢复。

全量快照，copy on write使得主线程在bgsave进程在保存快照时正常执行写操作。

![img](https://gitee.com/wanghengg/picture/raw/master/2020/a2e5a3571e200cb771ed8a1cd14d5558.jpg)

内存快照的缺点是很难取舍两次快照之间的时间间隔。

## redis主从库模式

Redis 提供了主从库模式，以保证数据副本的一致，主从库之间采用的是读写分离的方式。

**读操作：**主库、从库都可以接收；

**写操作：**首先到主库执行，然后，主库将写操作同步给从库。

![img](https://gitee.com/wanghengg/picture/raw/master/2020/809d6707404731f7e493b832aa573a2f.jpg)

主从库数据第一次同步的三个阶段：

![img](https://gitee.com/wanghengg/picture/raw/master/2020/63d18fd41efc9635e7e9105ce1c33da1.jpg)

**通过“主 - 从 - 从”模式将主库生成 RDB 和传输 RDB 的压力，以级联的方式分散到从库上。**

![img](https://gitee.com/wanghengg/picture/raw/master/2020/403c2ab725dca8d44439f8994959af45.jpg)

**主从库网络连接断了怎么办？**

当主从库断连后，主库会把断连期间收到的写操作命令，写入 replication buffer，同时也会把这些操作命令也写入 repl_backlog_buffer 这个缓冲区。**repl_backlog_buffer 是一个环形缓冲区**，主库会记录自己写到的位置，从库则会记录自己已经读到的位置。

![img](https://gitee.com/wanghengg/picture/raw/master/2020/13f26570a1b90549e6171ea24554b737.jpg)

## 主库挂了，如何不间断服务（哨兵机制）

哨兵其实就是一个运行在特殊模式下的 Redis 进程，主从库实例运行的同时，它也在运行。哨兵主要负责的就是三个任务：监控、选主（选择主库）和通知。

监控：定时发PING命令，需要回复，如果指定时间没有回复就认定连接挂掉。

选主：如果主库挂掉，哨兵就需要从多个从库里面按照一定规则选一个从库实例，把它作为新的主库。

通知：选择好新主库之后，哨兵把新主库的连接信息发给其他从库，让它们重新执行replicof命令，和新主库建立连接，并进行数据复制。同时，哨兵会把新主库的连接信息通知给客户端，让它们把请求操作发给新主库。

哨兵避免误判的解决办法：哨兵集群，多个哨兵进行判断（**少数服从多数原则**）。

![img](https://gitee.com/wanghengg/picture/raw/master/2020/1945703abf16ee14e2f7559873e4e60d.jpg)

### 如何选择新主库

筛选+打分

筛选没有断连并且网络状态良好的从库。

打分：我们可以分别按照三个规则依次进行三轮打分，这三个规则分别是**从库优先级、从库复制进度以及从库 ID 号**。只要在某一轮中，有从库得分最高，那么它就是主库了，选主过程到此结束。如果没有出现得分最高的从库，那么就继续进行下一轮。

## 切片集群

redis通过数据切片来避免单个数据库实例过大而产生的生成RDB时fork子进程导致主线程阻塞时间过长的问题。通过哈希槽，切片集群就实现了数据到哈希槽、哈希槽再到实例的分配。

重定向机制：

![img](https://gitee.com/wanghengg/picture/raw/master/2020/350abedefcdbc39d6a8a8f1874eb0809.jpg)

## string类型

![img](https://gitee.com/wanghengg/picture/raw/master/2020/ce83d1346c9642fdbbf5ffbe701bfbe3.jpg)

