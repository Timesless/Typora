### 1. 事务并发控制

事务并发控制保证隔离性

#### 1.1 事务

数据库提供增删改查基础操作，用户可以灵活组合以实现复杂的语义，用户希望一组操作做为整体生效，这就是事务。

特性：原子性，隔离性，持久性是为了保证一致性



#### 1. 2 2PL

悲观并发控制唯一实现2PL（Two-phase Locking二阶段锁）

> 阶段一：Growing，事务向锁管理器请求它需要的所有锁（存在加锁失败的可能）。
>
> 阶段二：Shrinking，事务释放Growing阶段获取的锁，不允许再请求新锁。
>
> **SS2PL**：只能在事务结束后释放锁，杜绝读未提交



#### 1.3 TO

乐观并发控制唯一实现 TO（Timestamp Ordering）

三种实现：

+ Basic T/O
+ OCC（Optimistic Concurrency Control）

> OCC分3阶段
>
> + Read Phase （对于读，放Read Set，对于写，写入临时区放入Write Set，属于未提交结果，其他事务读取不到【这与MVCC不同】）
> + Validation Phase，重扫Read Set，Write Set，检验数据是否满足Isolation Level，如果满足Commit否则Abort
> + Write Phase，或者叫做Commit Phase，把临时区数据更新到数据库中，完成事务提交

+ MVCC

>  **Multi-Version可看作另一个维度， 可以应用在任意算法上，形成MV-TO, MV-2PL, MV-OCC**
>
>  MVCC为每条记录维护多个快照（Snapshot，物理版本），通过起止时间戳（Begin / End Timestamp）维护副本可见性
>
>  Read：通过起止时间戳判定记录是否对当前事务可见（OCC读不到未提交的记录，不需要判断）
>
>  Update： 创建一个新版本（Version）
>
>  Delete：更新End Timestamp
>
>  **通过MVCC实现读写不阻塞，主流数据库几乎都采用该项优化技术**（并未完全解决事务并发）
>
>  为实现Serializable，MVCC通过不同方法实现
>
>  + 基于锁的MV-2PL，如Mysql
>
>  + 基于TO的MV-TO，如PostgreSQL，准确来说PG实现为SSI（Serializable Snapshot Isolation），不算MV-TO
>  + 基于OCC的乐观算法的，MV-OCC。即：读写时不验证，延迟到提交时验证



#### 1.4 意向锁

Intention Lock

> 层次越高的锁（表锁），可以有效减少对资源的占用，显著减少锁检查的次数，但会严重限制并发。层次越低的锁（行锁，对OLTP最小操作单元是行），有利于并发执行，但在事务请求对象多的情况下，需要大量的锁检查。数据库系统为了解决高层次锁限制并发的问题，引入了意向（Intention）锁的概念
>

+ Intention-Shared (IS)：表明其内部一个或多个对象被S-Lock保护，例如某表加IS，表中至少一行被S-Lock保护。

+ Intention-Exclusive (IX)：表明其内部一个或多个对象被X-Lock保护。例如某表加IX，表中至少一行被X-Lock保护。

+ Shared+Intention-Exclusive (SIX)：表明内部至少一个对象被X-Lock保护，并且自身被S-Lock保护。例如某个操作要全表扫描，并更改表中几行，可以给表加SIX

读者可以思考一下，为啥没有XIX或XIS



#### 1.5 InnoDB MVCC

Snapshot Read & Current Read（快照读 & 当前读）

``` sql
-- 快照读
SELECT * FROM table WHERE ?

-- 当前读（需要加锁）
SELECT * FROM table WHERE ? LOCK IN SHARE MODE （S锁）
SELECT * FROM table WHERE ? FOR UPDATE（X锁）
-- 包含当前读
INSERT INTO table VALUES(...)
UPDATE table SET ? WHERE ?
DELETE FROM table WHERE ?
```

> UPDATE table SET ? WHERE ？
>
> 当update SQL发送给MySQL后，MySQL Server根据where条件请求InnoDB读取第一条满足条件的记录，InnoDB返回记录并加锁（current read）。待MySQL Server收到加锁的记录后，再发起一个update请求更新这条记录，完成后再取下一条记录
>
> **由此update包含一个当前读，同理delete，insert**