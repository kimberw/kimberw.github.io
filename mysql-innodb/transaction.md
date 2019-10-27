# 事务

事务需符合 ACID 的特性：

- 原子性 atomicity ：保证事务只存在 做 和不做两种状态。
- 一致性 consistency ：保证数据库的完整性约束没有被破坏，事务在提交和回滚时。
- 隔离性 isolation ：保证并发时事务相互之间不可见
- 持久性 durability ：事务一旦提交结果就是永久性的。持久性保证数据的高可靠性。而高可用性还需系统共同配合。

## 事务分类

- 扁平事务 flat transactions
- 带有保存点的扁平事务 flat transactions with savepoints
- 链事务 chained transactions
- 嵌套事务 nested transactions
- 分布式事务 distributed transactions

## 事务的实现

事务隔离性由锁来实现，原子性，持久性由 redo log 来实现。 一致性由 undo log 来保证。

###### redo 恢复提交事务操作的页操作，通常是物理日志，记录页中物理修改操作。基本上是顺序写的。在数据库运行时不需要对redo log文件进行读取。

###### undo 回滚行记录到某个特定版本。是逻辑日志，根据每行记录进行记录。帮助事务回滚及MVCC的功能。在数据库运行时需要进行随机读写。

### redo

#### 概念

重做日志用来实现事务的持久性，由两部分组成：1，内存中的重做日志缓冲 redo log buffer 是易失的。2，重做日志文件 redo log file 是持久的

InnoDB 采用 Force log at commit 机制实现事务的持久性。 

为了确保每次重做日志的写入，每次commit时，将重做日志缓冲写入文件系统缓冲，再调用fsync操作，写入磁盘文件。所以fsync的效率，及磁盘性能决定了数据库的性能。

参数 innodb_flush_log_at_trx_commit 用来控制重做日志刷新到磁盘的策略，默认是1，表示每次commit必须调用一次fsync。0，表示事务提交时不自动调用fsync，等待master thread来定时完成。2，表示事务提交仅将重做日志写入文件系统缓冲中。不进行fsync操作。此时若数据库发生宕机恢复时不会导致事务丢失。若操作系统宕机则会丢失部分事务。

对比二进制日志 binlog，用来进行point-in-time的恢复及主从复制（replication）环境的建立，但是binlog 与 redo log 本质区别

|                | binlog                      | redo log                                                     |
| -------------- | --------------------------- | ------------------------------------------------------------ |
| 数据产生源     | mysql数据库上层产生         | InnoDB 存储引擎层产生                                        |
| 内容形式       | 逻辑日志，记录对应的sql语句 | 物理日志，记录对于每个页的修改                               |
| 磁盘写入时间点 | 事务提交后进行一次写入      | 在事务进行中不断被写入，多个事务同时执行时会存在日志记录的穿插 |

#### log block

重做日志以512字节进行存储。所以其缓存及日志文件都是以块的形式保存。重做日志块 redo log block 每块512字节。大小同磁盘扇区一样，所以重做日志的吸入可以保证原子性，不需要 doubelwrite。

#### log group

每个group 含有多个重做日志文件，源码支持，但是实际只有一个log group。

InnoDB 存储引擎在运行过程中，log buffer根据一定的规则将内存中的log block刷新到磁盘。

- 事务提交时
- 当log buffer中一半的内存空间已经被使用时
- log checkpoint 时

当一个redo log file 被写满时，会接着写下一个redo log file，循环使用 round-robin。

redo log file 除了保存log buffer 刷新到磁盘的 log block，还保存其他信息。即每个redo log file 前2k不保存log block。每个log group 的第一个log file 会占用着2k空间。其他log file 保留空间不使用。

#### LSN

LSN 是 log sequence Number。日志序列号，在InnoDB
中占用8字节，单调递增。表示：

- 重做日志写入的总量：记录的是重做日志写入的字节数
- checkpoint的位置：log flushed up to 表示刷新到重做日志文件的LSN last checkpoint at 表示刷新到磁盘的LSN
- 页的版本：每个页头部记录 FILE_PAGE_LSN，当页最后刷新时LSN的大小。以此来判断该页是否需要恢复操作。

#### 恢复

由于重做日志是物理日志，比 逻辑日志 恢复速度快很多。InnoDB存储引擎自身页对恢复进行一定程度的优化：顺序读取，并行应用重做日志。

仅需要恢复发生宕机后，即（checkpoint大于某值）的日志部分。

重做日志是物理日志，是幂等的。

二进制日志不是幂等的。

### undo

#### 概念

undo用来实现事务回滚操作。undo存放在数据库内部的一个特殊段 segment 中，称为 undo段，位于共享表空间内。undo是逻辑日志。因为数据库是多并发的，不能讲当前页直接回滚到事务开始前的样子，这样会影响多其他事务。所以undo log 是逻辑日志。回滚操作实际上是将undo log 中的操作做相反的工作。

除了回滚，undo 还被用来实现 MVCC。InnoDB 在读取一行记录是，若该记录被其他事务占用，则可以通过undo 读取之前的行版本信息来实现非锁定读取。

undo log 会产生 redo log，因为 undo log 页需要持久性的保护。

#### undo 存储管理

InnoDB有 rollback segment，每个 rollback segment 记录了 1024个 undo log segment。每个 undo log segment 中进行 undo 页的申请。目前InnoDB 支持最大 128个rollback segment。

在事务提交时，InnoDB会做两件事情：

- 将 undo log 放入列表中，以供之后的purge操作。
- 判断 undo log 所在的页是否可以重用，若可以分配给下一个事务使用。

事务提交后，并不能马上删除undo log 及 undo log 所在的页，因为 MVCC 可能会访问到行记录的之前版本。故事务提交时将 undo log 放入一个链表中，是否可以删除由purge线程来判断。

InnoDB 并非为每个事务操作分配一个单独的 undo 页。这样若事务操作频繁时，purge 的频率将跟不上undo页的新建。所以当事务提交时，将undo og 放入链表中，然后判断undo 页的使用空间是否小于 3/4，若是则可以被重用，新的undo log 记录在当前undo log 之后。

由于存放 undo log 的列表是以记录进行组织的，而 undo 页可能存放着不同事务的 undo log，因此purge操作需要设计磁盘的离散读取，是较缓慢的过程。

#### undo log 格式

undo log 分为：

- insert undo log
- update undo log

insert操作只对事务本身可见，对其他事务不可见。所以 undo log 可以在事务提交后直接删除。不需要进行 purge 操作。update undo log 记录的是对 delete 和 update 操作产生的undo log。该log 需要提供 MVCC机制，因此不能再事务提交时进行删除。而是放入 undo log 链表，等待 purge 线程进行最后的删除。

#### undo 类型

- TRX_UNDO_INSERT_REC 插入记录
- TRX_UNDO_UPD_EXIST_REC 更新 non-delete-mark 记录
- TRX_UNDO_UPD_DEL_REC 将 delete 记录标记为not delete
- TRX_UNDO_DEL_MARK_REC 将记录标记为delete

### Purge 

对于delete update操作，并不直接删除原有的数据，而是将聚集索引中delete flag 设置为1，并没有被删除。此时辅助索引上的记录同样存在，甚至没有产生undo log。而真正的删除操作被延迟在purge 操作中完成。

InnoDB存储引擎先从 history list 中找 undo log，然后再从 undo page 中找其他的 undo log，避免大量的随机读取操作。 

可以通过修改全局动态参数 innodb_purge_batch_size 来控制每次 purge 操作需要清理的 undo page 数量。若设置越大，则可供重用的undo page 越多，减少了磁盘存储空间和分配的开销，但是设置过大，会使得每次purge处理更多的 undo page，从而导致CPU和磁盘IO 过于集中，使性能下降。

若InnoDB存储引擎的压力非常大时，并不能高效的进行 purge 操作。那么 history list 的长度会变得越来越长，全局动态参数 innodb_max_purge_lag 用来控制 history list 长度。若长度大于该参数，将延缓DML的操作。delay 是针对行进行的，若某DML操作5行数据，则被延迟时间为 5*delay。可以通过修改 innodb_max_purge_lag_delay 来控制最大delay时间。

### group commit

若事务非只读事务，则每次事务提交时都需进行一次fsync操作，以保证重做日志都已经写入磁盘。确保数据持久性。所以数据库的性能较依赖于磁盘fsync性能。所以数据库提供一个group commit的方式，通过一次fsync可以刷新多个事务日志到文件中。

1. 修改内存中事务对应的信息，并将日志写入重做日志缓冲。
2. 调用fsync将确保日志都从重做日志缓冲写入磁盘。该步骤可以通过group commit 来提高新能。

旧版mysql与InnoDB无法在开启二进制日志的同时打开group commit。并且在线环境多使用 replication 环境。因此二进制日志的选项基本都是开启的。原因是：由于备份及恢复的需要，需要保证mysql提交的二进制日志的写入顺序和InnoDB层的事务提交顺序一致。mysql内部使用 prepare_commit_mutex 锁，因此下述步骤3.1不能在其他事务进行3.2的时候同时进行。

1. 当事务提交时InnoDB存储引擎进行prepare操作。
2. mysql 数据上层写入二进制日志
3. InnoDB存储引擎将日志写入重做日志文件
   1. 修改内存中事务对应的信息，将日志写入重做日志缓冲
   2. 调用 fsync 将确保日志都从重做日志缓冲写入磁盘

新版mysql采用 Binary Log Group Commit（BLGC）。mysql 数据库提交时首先按顺序将其放入一个队列中，队列的第一个事务称为leader其他为follower，leader控制着follower的行为。

- flush 阶段，将每个事务的二进制日志写入内存中
- sync 阶段，将内存中的二进制日志刷新到磁盘，若队列中有多个事务，仅一次fsync操作完成二进制日志的写入
- commit 阶段，leader 根据顺序调用存储引擎层事务的提交，支持group commit

### 对事务操作的统计

tps = (com_commit+com_rollback)/time 要求所有事务必须显示提交。

### 事务的隔离级别

- READ UNCOMMITTED
- READ COMMITTED
- REPEATABLE READ
- SERIALIZABLE

InnoDB默认级别是REPEATABLE READ。采用 Next-Key Lock 来避免幻读达到 SERIALIZABLE级别。同时**性能并不低**

可能引起主从不一致的两个问题，及解决方式：

- 在read committed 下，事务没有使用 gap lock 。可以使用read repeatable级别。
- statement 格式记录的是 master 上产生的sql 语句，因此 master 执行顺序 和statement 格式中记录顺序可能出现不一致。 可以使用row 格式的二进制日志文件。

## 分布式事务

InnoDB 存储引擎支持XA事务，实现对分布式事务的支持。对于多个独立的事务资源参与到一个全局的事务中，要么都提交要么都回滚。

XA 事务由一个或多个资源管理器（Resource Managers）、一个事务管理器（Transaction Manager）以及一个应用程序（Application Program）组成

- 资源管理器：提供访问事务资源的方法，通常一个数据库就是一个资源管理器
- 事务管理器：协调参与全局事务中的各个事务，需要和参与全局事务的所有资源管理器通信
- 应用程序：定义事务的边界及指定全局事务的操作

分布式事务采用两段式提交（two-phase commit）：

1. 第一阶段：所有参与全局事务的节点都开始准备 prepare，告知事务管理器他们准备提交了
2. 第二阶段：事务管理器告诉资源管理器执行 ROLLBACK 还是 COMMIT。

### 内部XA事务

在 mysql 数据库中还存在另外一种分布式事务，其在存储引擎与插件之间，或者存储引擎与存储引擎之间。称为内部XA事务。常见的是binlog与InnoDB存储引擎之间。

由于复制需要，大多数数据库开启了binlog。在事务提交时，先写binlog再写重做日志，分别都是原子操作。但是若中间过程宕机了，会导致复制后主从不一致问题。所以在binlog和redo log之间采用XA事务，即在事务提交时，InnoDB先做一个 PREPARE操作，将事务xid写入，然后写binlog，若在写redo log 之前宕机了，则数据库重启或者复制后，先检查UXID事务是否已经提交，若没有则再进行一次提交。

## 不好的事务习惯

- 在循环中提交

  1，发生错误时，数据库可能停留在一个未知位置。2，性能问题，redo log只需写一次 做一次fsync即可。

- 使用自动提交

  1，不方便做事务统计，（自动提交不记录）2，不同语言API的自动提交不同。所以干脆将提交操作交给开发人员管理。避免出现没有意识到的问题

- 使用自动回滚

  不清楚自动回滚时事务是否正确执行亦或直接回滚。并且不清楚事务出错回滚的原因。所以将自动回滚关闭，将之交给开发人员把握

## 长事务

执行时间较长的事务，例如定时更新账户利息，对应账户数量较大，执行较久。一般将长事务转化为小批量（mini batch）事务来处理。避免事务进行中，由于数据库，操作系统或者硬件等发生问题，需要重新开启事务带来的巨大代价。将执行结果放入 batchcontext表中，来记录当前事务执行状态。避免重复，又可以查看事务进行进度。在操作利息时，人为的加一个share lock，避免过程中数据被更新修改。