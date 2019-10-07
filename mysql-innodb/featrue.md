# 特性

## 插入缓存 (Insert Buffer)

较聚集索引而言，非聚集索引的插入是离散的非顺序的。随机的插入索引页会导致低效率。

### Insert buffer 需满足两个条件：

1. 索引是辅助索引
2. 索引不是唯一的

### 简单描述：

​	对于非聚集索引的插入或更新操作，不是每次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，直接插入。若不在，则先放在一个 Insert Buffer 对象中，数据库则认为已经完成了插入，其实不然，只是存放在另一个位置。然后再以一定的频率和情况进行 Insert Buffer 和非聚集索引页子节点的 merge 操作。通常可以将多个插入合并到一个操作中，从而大大提高非聚集索引插入的性能。

### Change Buffer：

可以将其视为 Insert Buffer 的升级。分别为： Insert Buffer、 Delete Buffer、 Purge Buffer

#### 对于一条记录进行 UPDATE 操作可能分为两个过程：

1. 将记录标记删除
2. 真正将记录删除

Delete Buffer 对应第一个过程，标记删除， Purge Buffer 对应第二个过程，真正删除。

### 内部实现

Insert Buffer 的数据结构是一颗 B+ 树。

### Insert Buffer 合并时机

1. 辅助索引页被读取到缓冲池时
2. Insert Buffer Bitmap 页追踪到该辅助索引页已无可用空间时
3. Master Thread 定期

## 两次写 (Double Write)

数据页可靠性保障。针对部分写失效导致数据丢失的场景。

在对缓冲池中的脏页进行刷新时，并不直接写磁盘。而是会通过 memcpy 函数将脏页先复制到内存中的 doublewrite buffer ，之后通过 doublewrite buffer 再分两次 每次1MB顺序的写入共享表空间的物理磁盘上。并马上调用fsync函数，同步磁盘。避免缓冲写带来的问题。

若操作系统在将页写入磁盘过程中发生错误，在回复过程中，InnoDB 存储引擎可以从共享表空间中的 doublewrite 中找到该页的副本，将其复制到表空间文件。再应用重做日志进行恢复。

一般只在从服务器（slave server）时，为了快的性能关闭 doublewrite 功能。主服务器（master server）建议保持开启状态。

## 自适应哈希索引 (Adaptive Hash Index) AHI

hash 一般仅需一次查找就能定位数据，B+树的查找次数取决于其树的高度，一般为3~4层，即3~4次查询。

InnoDB 存储引擎会监控对表上各索引页的查询，并在必要时自动增加 AHI。

### 添加限制

- 以类似 where a=xxx 或者 where a=xxx and b=xxx 其中一种模式连续访问100次
- 页通过上述模式访问N次，其中 N=页中记录*1/16

## 异步IO (Async IO)

为了提高磁盘操作性能，采用 AIO的形式处理磁盘操作。

- AIO不需要用户等待 磁盘IO完成后才可以继续。用户可以发出多个IO请求。然后等待IO完成。
- AIO的另一个优势是可以进行 IO Merge 操作。即是将同一批次的多个 IO 请求，判断其对应页是连续的则可以连续读多个页。

目前 InnoDB 已支持内核级别的 AIO ，称为 Native AIO。 Native AIO由操作系统提供支持，linux 和 windows 都已支持，MacOS还未提供。官网的测试显示，启用 Native AIO 速度可提高75%。在 InnoDB 引擎中，read ahead 和脏页的刷新（磁盘的写入操作）均由AIO完成。

## 刷新临接页 (Flush Neighbor Page)

当刷新一个脏页时，InnoDB 会检测该页所在区（extent）的所有页，如果存在脏页，则一起进行刷新。通过AIO可以将多个IO写入操作合并为一个IO操作。该机制在机械磁盘下有显著效率提升。

### 存在问题

- 是否可以不对不怎么脏的页进行操作，该页很可能在刷新后很快变成脏页。
- 固态磁盘拥有较高的IOPS，此特性优化不明显，同时可能带来新的问题（如上一条）