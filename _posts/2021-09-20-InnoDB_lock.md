---
layout: post
title: MySQL InnoDB 存储引擎中的锁
categories: MySQL
tags: [MySQL]
---
## 为什么数据库中需要锁？

对于数据库这种基础应用，开发者当然是希望通过最大程度地利用数据库的并发访问从而带来性能上的提升；但是并发就会带来数据一致性、隔离性等问题。

数据库使用锁的目的就是为了支持对共享资源进行并发访问，提供数据的完整性和一致性。

## InnoDB 中的锁

对于锁的概念这里就不做过多的介绍了，接触过 Java 并发编程的想必都会有所了解。

本文重点关注表数据上的锁，对于缓冲池中LRU列表的操作锁不做详细解释。

MySQL 中锁的种类很多，有常见的表锁和行锁，表锁是对一整张表加锁，虽然可以分为读锁和写锁，但毕竟还是锁住整张表，会导致并发能力下降，一般是做 DDL 处理时使用；行锁则是锁住数据行，只锁住有限的数据，对于其它数据不加限制，所以并发能力强。

InnoDB 存储引擎会在行级别上对表数据上锁。

InnoDB 中的锁相较于其他语言中接触到的锁不太一样，接下来通过 InnoDB 锁的类型展开介绍：

### 锁的类型

InnoDB 存储引擎实现了两种标准的行级锁：

* 共享锁（S Lock），允许事务读一行数据
* 排他锁（X Lock），允许事务删除或更新一行数据

|        | 排他锁 | 共享锁 |
| ------ | ------ | ------ |
| 排他锁 | 不兼容 | 不兼容 |
| 共享锁 | 不兼容 | 兼容   |

从上表可发现 排他锁 与任何锁都不兼容，共享锁仅和共享锁兼容。即某一行加上共享锁之后，下一个会话只能加共享锁；加上排他锁之后，不能再加任何类型的锁。

为了支持在不同粒度上进行加锁操作，InnoDB 存储引擎支持一种额外的锁方式，称之为意向锁（Intention Lock）。意向锁是表级别的锁，设计目的主要是为了在一个事务中揭示下一行将被请求的锁类型。同样也存在两种意向锁：

* 意向共享锁（IS Lock），事务想要获得一张表中某几行的共享锁
* 意向排他锁（IX Lock），事务想要获得一张表中某几行的排他锁。

简单来说，引入意向锁其实就是为了表示是否有其他请求锁定表中的某一行数据。如果没有意向锁，某一行添加了互斥锁的同时，另一个请求要对全表进行修改，就需要扫描全表是否有行被锁定，这种情况下效率非常低；引入意向锁之后，添加行锁前，会先为表添加意向锁，不需要全表扫描，只需要等待其他意向锁释放即可。

### 事务的加锁方式

讲到数据库中的锁，不可避免地需要涉及到事务的隔离级别，下面简单介绍 InnoDB 事务的四种隔离级别：

| 隔离级别                     | 脏读（Dirty Read） | 不可重复读（NonRepeatable Read） | 幻读（Phantom Read） |
| ---------------------------- | ------------------ | -------------------------------- | -------------------- |
| 未提交读（Read uncommitted） | 可能               | 可能                             | 可能                 |
| 已提交读（Read committed）   | 不可能             | 可能                             | 可能                 |
| 可重复读（Repeatable Read）  | 不可能             | 不可能                           | 可能                 |
| 串行化（Serializable）       | 不可能             | 不可能                           | 不可能               |

在数据库操作中，为了有效保证并发读取数据的正确性，提出的事务隔离级别。数据库锁，也是为了构建这些隔离级别存在的。

* **未提交读（Read UnCommitted）**: 可能会读取到其他会话中未提交事务修改的数据（数据库一般都不会用）
* **提交读（Read Committed）**：只能读取到已经提交的数据
* **可重复读（Repeatable Read）**：在同一个事务内的查询都是事务开始时刻一致的，InnoDB 默认级别。
* **串行读（Serializable）**：完全串行的读，每次读都需要获得表级共享锁，读写相互都会阻塞。

### 锁问题

因为事务隔离性的要求，锁会带来一些问题：

* **脏读**：在不同的事务下，当前事务可以读到另外事务未提交的数据，即脏数据，违反了数据库的**隔离性**。脏读现象在生产环境中并不常发生，因为需要事务的隔离级别为 Read UnCommitted
* **不可重复读**：在一个事务内多次读取同一数据集合。在第一个事务中的两次读数据之间，由于第二个事务的修改，导致第一个事务两次读到的数据可能是不一样的。不可重复读和脏读的区别是：脏读是读到未提交的数据，而不可重复读读到的却是已经提交的数据，违反了数据库事务一致性的要求。
* **幻读**：在同一事务下，连续执行两次同样的 SQL 语句可能导致不同的结果，第二次的 SQL 语句可能会返回之前**不存在**的行。

上述**不可重复读**和**幻读**的定义很相似，容易搞混。其实重点在于幻读定义中的“不存在的行”概念，不可重复读重点在于 update 和 delete（读取了其他事务更改的数据），幻读的重点在于 insert（读取了其他事务新增的数据）。

如果使用锁机制来实现这两种隔离级别，在可重复读中，该 SQL 第一次读取到数据后，就将这些数据加锁，其他事务无法修改这些数据，就可以实现可重复读了。但是行锁无法锁住 INSERT 的数据（新增），下一次读取就会发现莫名其妙多了一条之前没有的数据，需要 Serializable 隔离级别，读锁和写锁互斥，但会极大的降低数据库的并发能力。

但是在 InnoDB 存储引擎，采用 **Next-Key Locing** 的算法可以避免幻读问题。

### MVCC 在 InnoDB 中的实现

为了减少锁处理的时间，提升并发能力，InnoDB 中使用了“一致性的非锁定读（consistent nonlocking read）”技术。**一致性非锁定读**是指 InnoDB 存储引擎通过行多版本控制（multi versioning）的方式来读取当前执行时间数据库中行的数据。如果读取的行正在执行 DELETE 或 UPDATE 操作，这时读取操作不会因此去等待行上锁的释放，而是去读取行的一个**快照数据**。

快照数据是指该行的之前版本的数据，一个行记录可能有不止一个快照数据，一般称这种技术为行多版本技术。由此带来的并发控制，称之为多版本并发控制（Multi Version Concurrency Control，MVCC）。在 InnoDB 存储引擎中，快照数据的实现通过 undo 段来完成。undo 用来在事务中回滚数据，因此快照数据本身是没有额外的开销的。

在事务隔离级别 Read Committed 和 Repeatable Read 下，InnoDB 存储引擎使用非锁定的一致性读。但两者对于快照数据的定义是不同的，在 Read Committed 事务隔离级别下，对于快照数据，非一致性读总是读取被锁定行的**最新一份快照数据**；而在 Repeatable Read 事务隔离级别下，对于快照数据，非一致性读总是读取**事务开始时的行数据版本**。

与 “一致性非锁定读” 相对应的是 “一致性锁定读”，用户**显式**地对数据库读取操作进行加锁以保证数据逻辑的一致性。

InnoDB 存储引擎对于 SELECT 语句支持一致性的锁定读操作：

* SELECT ... FRO UPDATE (添加行X锁)
* SLECT ... LOCK IN SHARE MODE（添加行 S 锁）

### 锁的算法

InnoDB 存储引擎有 3 种行锁的算法：

#### 记录锁（Record Lock）

事务加锁后锁住的只是表的某一条记录。

**记录锁出现条件**：精准条件命中，并且命中的条件字段时唯一索引。因为在 InnoDB 行锁是加到索引记录上的。

**记录锁的作用**：避免数据在查询的时候被修改的重复读问题；避免在修改的事务未提交前被其他事务读取的脏读问题。

#### 间隙锁（GAP Lock）

间隙锁，锁定一个范围，但不包含记录本身

**出现的条件**：使用普通索引锁定；使用多列唯一索引；使用唯一索引锁定多行记录

**作用**：防止幻读

#### 临键锁（Next-Key Lock）

Gap Lock + Record Lock，锁定一个范围，并且锁定记录本身

