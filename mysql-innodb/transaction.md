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

