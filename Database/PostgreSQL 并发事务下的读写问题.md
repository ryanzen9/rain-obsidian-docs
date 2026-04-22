---
title: PostgreSQL 并发事务下的读写问题
date: 2026-04-21
author: Ryan Zeng
tags:
  - 数据库
  - ERP
categories: []
draft: false
---

## 一、并发事务下的读问题

在讲解隔离级别之前，我们必须先清楚，如果没有任何隔离，并发事务下的读操作会带来哪些问题。主要有以下三种：

### 1. 脏读 (Dirty Read)

- **比喻：** 我正在写一份草稿，还没写完，你就拿去看，并基于我的草草稿内容去做了决策。结果我后来发现草稿里有重大错误，撕了重写，你的决策就建立在了错误的信息之上。

- **定义：** 一个事务读取到了另一个事务**尚未提交**的数据。事务在这其中并不是相互隔离的。

- **重点：** 读到了并不存在的数据。

### 2. 不可重复读 (Non-Repeatable Read)

- **比喻：** 你去商店问一台电视的价格，店员告诉你 5000 元。你去取钱的几分钟里，店员把价格改成了 5500 元。你回来再问，发现价格变了。在你的这次"购物"事务中，同一个商品的价格不可复现了。

- **定义：** 一个事务内，多次读取**同一行数据**，得到的结果却不一致。重点在于同一条数据在一个事务中前后的查询结果可能会出现不一致。

- **重点：** 并发事务中数据一致性的干扰。

### 3. 幻读 (Phantom Read)

- **比喻：** 你在点名，数了一下教室里有 30 个人。这时，在你没注意的时候，悄悄从后门溜进来一个学生。随即登记到达人数全部为 30 人时发现记入了 31 人。

- **定义：** 一个事务内，多次执行**相同的范围查询**，得到的结果集记录数不一致。当其他事务出现插入或者删除操作时，即使有行锁也不能保证事务内的查询集数量是一致的。

- **重点：** 并发事务中数据集数量不同的干扰。

## 二、四种隔离级别详解

### 1. 读未提交 (Read Uncommitted)

**技术说明：**  
这是最低的隔离级别。它允许一个事务读取到另一个事务尚未提交的更改。这种级别下，并发性能最高，但数据一致性最差。由于会产生脏读，在实际生产环境中基本不会使用。在这种情况下事务间的读操作并不是相互隔离的。

**场景介绍：**  
某数据分析平台需要对一个巨大的交易表进行实时（但允许有微小误差）的统计汇总，比如计算当前小时的总交易额。为了追求极致的性能，不希望统计查询被任何正在进行的交易长时间阻塞。

几乎所有数据库都支持，但很少使用。

> 需要注意的是 PG 虽然接受 UNCOMMITTED ，但它实际上会把 `READ UNCOMMITTED` **按 `READ COMMITTED` 处理**。也就是说，PG 不支持真正的脏读。

**举例（以 MySQL 为例）：**

```sql
-- 终端 A
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
UPDATE accounts SET balance = 900 WHERE id = 1; -- 账户余额从1000减到900，但不提交

-- 终端 B
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1; -- 查询结果是 900 (脏读)
COMMIT;

-- 终端 A
ROLLBACK; -- 终端A回滚事务

-- 再次在终端 B 查询
SELECT balance FROM accounts WHERE id = 1; -- 结果变回 1000
```

### 2. 读已提交 (Read Committed)

**技术说明：**  
一个事务只能读取到已经提交的事务所做的修改。这个级别解决了"脏读"问题。这是 PostgreSQL 的默认隔离级别。

> 同样拥有读一致性，但它是**语句级**的一致性，不是**事务级**的一致性。
> 单条语句：每条 `SELECT` 只能看到**该语句开始前**已经提交的数据；不会看到未提交数据，也不会看到并发事务在该语句执行过程中提交的变化。也就是说，**单条语句内部是一致的**。
> 事务：即使它们在同一个事务里，因为其他事务可能在这两条语句之间提交了更新。因此不能保证整个事务期间所有读都一致。也就是 **不可重复读** 问题。

**解决脏读：**  
对读操作不加锁，使用了 MVCC 机制来避免脏读，但是时机在于每一次写操作都会重新创建一次快照，然后对写操作后的数据进行行锁。但是创建的快照在事务的进行过程随时读取的，可能前后的某条数据是不一致的。快照的生命周期是语句级别的。

**场景介绍：**  
一个电商网站，用户正在下单。下单过程是一个事务，需要先读取商品库存，再创建订单。与此同时，另一个用户可能正好完成了对同一个商品的购买并付款成功。

**举例 (以 PostgreSQL 为例):**

```sql
-- 终端 A
BEGIN; -- PostgreSQL默认就是READ COMMITTED
SELECT stock FROM products WHERE id = 'P001'; -- 假设返回 1

-- 终端 B
BEGIN;
UPDATE products SET stock = 0 WHERE id = 'P001';
COMMIT; -- 提交事务

-- 终端 A
-- 在同一个事务中再次查询
SELECT stock FROM products WHERE id = 'P001'; -- 结果是 0,继续扣除会发生错误
COMMIT;
```

>**MVCC（Multi-Version Concurrency Control，多版本并发控制）** 是数据库系统中用来实现高并发和事务隔离级别的一种技术。
>在传统的锁机制中，如果一个事务正在读取数据（加读锁），另一个想要修改该数据的事务就会被阻塞；反之亦然。而 MVCC 的核心思想是：**“读不阻塞写，写不阻塞读”**。它通过为数据保留多个历史版本，让读操作读取旧版本的数据，从而避免了读写冲突，极大地提升了数据库的并发性能。

### 3. 可重复读 (Repeatable Read)

**技术说明：**  
保证在同一个事务中，多次读取同样记录的结果是一致的。这个级别解决了"不可重复读"的问题。这是 MySQL InnoDB 引擎的默认隔离级别。

**场景介绍：**  
一个银行系统在做一个复杂的日终结算。结算事务需要多次读取某个用户的账户余额来进行不同的计算（例如，计算利息、扣除年费）。在此期间，不能因为其他事务的存取款操作而影响本次结算中账户余额的一致性。

**解决不可重复读：**  
使用了 MVCC 机制 + 事务前一次快照来避免不可重复读，然后对写操作后的数据进行行锁。事务的执行前后读取的数据版本都是一致的。快照的生命周期是事务级别的。

**举例 (以 PostgreSQL 为例):**

```sql
-- 终端 A
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM employees WHERE department_id = 5; -- 返回20条（快照）

-- 终端 B
BEGIN;
INSERT INTO employees (name, department_id) VALUES ('小王', 5);
COMMIT;

-- 终端 A
-- 同一事务内再次查询（基于初始快照）
SELECT * FROM employees WHERE department_id = 5; -- 仍然是20条

-- 执行更新（对当前可见行加锁并尝试更新）
UPDATE employees
SET status = 'calculated'
WHERE department_id = 5;
-- 在 PostgreSQL 中，这里只会更新最初快照中可见的20条
-- 返回：UPDATE 20

-- 再次查询（仍基于同一快照）
SELECT * FROM employees WHERE department_id = 5; -- 仍然是20条

COMMIT;
```

> **PostgreSQL:** 它的可重复读实现与 MySQL 类似，也是基于 MVCC。快照是在**事务开始**时创建的，并且整个事务期间都使用这一个快照。通过 `xmin/xmax`的快照可见性判断实现隔离。

### 4. 可串行化 (Serializable)

**技术说明：**  
最高的隔离级别。它通过强制事务串行执行，即一个接一个地执行，来避免了前面提到的所有并发问题。对与读操作加上读锁（共享锁）。当其他事务要对有读锁的行进行操作时因为有了读锁不能添加排他锁，因此必须等待。

> **共享锁的特点：**
>
> - **读读兼容：** 其他事务也可以对这些行加共享锁（即也可以 SELECT ... LOCK IN SHARE MODE），大家可以一起读。
>
> - **读写互斥：** **任何事务都不能对这些加了共享锁的行进行修改 (UPDATE, DELETE) 或者加排他锁 (FOR UPDATE)。**

**场景介绍：**  
在一个金融或库存管理系统中，进行"秒杀"或分配唯一资源（如优惠券码）的操作。绝对不能出现两个用户同时抢到最后一个商品，或者领到同一个优惠券码的情况。

**解决幻读：**  
对事务开始后的查询进行读锁，在这个相关范围内阻塞同样范围的写操作。事务就像在一个静态的、被冻结的环境中工作。

**举例 (MySQL/PostgreSQL 通用):**

```sql
-- 终端 A
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
SELECT * FROM products WHERE stock > 0; -- 此时会对所有满足条件的行加读锁

-- 终端 B
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
-- 下面的插入操作会因为终端A的范围锁(间隙锁)而被阻塞，直到终端A提交或回滚
INSERT INTO products (name, stock) VALUES ('新商品', 10);
```

**引擎实现简介：**

- **MySQL (InnoDB):** 在 REPEATABLE READ 的基础上，会将所有的普通 SELECT 语句自动转换为 SELECT ... LOCK IN SHARE MODE，即给读取的行加上读锁（共享锁）。当其他事务尝试修改这些行时，必须等待。

- **PostgreSQL:** 它使用一种更先进的技术叫做 **SSI (Serializable Snapshot Isolation)**。它比传统的加锁方式更乐观，允许多个事务并发读写。但在事务提交时，它会检查是否存在可能破坏串行化执行的"读写依赖"，如果存在，则会主动让其中一个事务回滚失败，并提示用户重试。这种方式在无冲突时性能更好。

> **可串行化快照隔离 (Serializable Snapshot Isolation - SSI) (PostgreSQL)：**
>
> PostgreSQL 实现可串行化的方式更为"聪明"和"乐观"，它代表了现代数据库设计的趋势。
>
> **核心思想：** 我先乐观地让你并发执行，不随便加锁阻塞你。但是，在你要提交的时候，我会做一个"犯罪风险评估"，检查你在执行期间，有没有和其他事务形成"读写依赖环"，这种环路可能导致结果不再是串行的。如果发现有风险，就判你这次提交无效，让你回滚重试。
>
> 如果检测到这种危险的依赖关系，数据库会**主动让后提交的那个事务失败**，并抛出一个"串行化失败 (serialization failure)"的错误，要求应用程序捕获这个错误并重试整个事务。


## 三、并发事务下的写冲突

> 需要注意的是四种隔离级别所解决的问题都是并发事务下对 **读一致性** 的问题。 读一致性的成功并不一定意味着写一定可以成功。

>在 postgresql 中：读一致性由 MVCC 快照保证；写一致性由锁与并发冲突检测保证；当旧快照上的业务判断已经和当前真实行版本冲突时，数据库直接中止事务[40001]('https://www.postgresql.org/docs/current/mvcc.html?utm_source=chatgpt.com')。

首先要明确一个至关重要的点：

- **SELECT** 读是共享的，无论是哪个隔离级别，对读的操作都是共享的（可串行化会添加 s 锁，但依旧是读共享的）。

- **UPDATE** 都是"当前读"（Current Read）： 写入的是去读取数据库里**最新、最新、最新**的版本，对写的数据行进行行级 x 排他锁，如果其他事务同样需要写入该行，则会报错。

### 1. 锁等待死锁 (Lock-wait Deadlock)

如果两个事务互相等待对方持有的资源，就会形成死锁。除了读未提交，其他三种隔离级别都有可能出现。**可串行化：** 死锁的风险**更高**。因为它不仅写操作加锁，连读操作也加了共享锁。这就大大增加了锁的种类和持有时间，更多的锁交互自然带来了更高的死锁概率。

> **Postgres 的死锁检测与处理：**
> - 它并非全程阻塞，而是乐观地允许事务执行，同时跟踪它们之间的读写依赖关系。如果在事务准备提交时，发现其读写依赖关系构成了无法串行化的“环”，就会使其中一个事务失败，并报告“序列化失败”错误。

**场景：互相转账**

- **数据：** 账户 A 有 100 元，账户 B 有 100 元。

- **事务 A：** UPDATE accounts SET balance = balance - 10 WHERE id = 'A'; (锁住 A 行)

- **事务 B：** UPDATE accounts SET balance = balance - 10 WHERE id = 'B'; (锁住 B 行)

- **事务 A：** 接着想给 B 加钱：UPDATE accounts SET balance = balance + 10 WHERE id = 'B'; (尝试锁 B，但 B 被事务 B 锁住，等待...)

- **事务 B：** 接着想给 A 加钱：UPDATE accounts SET balance = balance + 10 WHERE id = 'A'; (尝试锁 A，但 A 被事务 A 锁住，等待...)

此时，A 在等 B 释放锁，B 在等 A 释放锁，形成死锁。

以 PostgreSQL 为例：

```sql
DROP TABLE IF EXISTS accounts;

CREATE TABLE accounts (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    balance NUMERIC(10, 2)
);

INSERT INTO accounts (id, name, balance) VALUES
(1, 'Alice', 1000.00),
(2, 'Bob', 2000.00);

-- =========================
-- 会话 A
-- =========================
BEGIN;

-- 步骤 1：锁住 id=1
UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

-- （切换到会话 B 执行其前两步）

-- 步骤 3：尝试更新 id=2（如果 B 已锁住，将阻塞）
UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

COMMIT;

-- =========================
-- 会话 B
-- =========================
BEGIN;

-- 步骤 2：锁住 id=2
UPDATE accounts
SET balance = balance - 200
WHERE id = 2;

-- （切换回会话 A 执行其第二个 UPDATE，使其进入等待）

-- 步骤 4：尝试更新 id=1（触发死锁检测）
UPDATE accounts
SET balance = balance + 200
WHERE id = 1;

COMMIT;

-- =========================
-- 结果说明
-- =========================
-- PostgreSQL 会检测死锁并终止其中一个事务：
-- ERROR: deadlock detected
-- 被终止的事务需要执行：
-- ROLLBACK;
-- 另一个事务可以正常 COMMIT;
```

### 2. 更新丢失（应用层覆写）

在业务代码中开启事务 A 执行 `SELECT *` 加载入内存中进行业务逻辑的计算。事务 B 在 A 读取之后更新相同的列，可能被你带着旧值一起覆盖回去，从而看起来像事务 B 的更新丢失了。

**场景：秒杀抢购最后一件商品**

- **数据：** stock = 1

- **事务 A：** 读取到 stock=1，判断可以购买。执行 UPDATE stock = 0 ...，抢到锁，成功。

- **事务 B：** 也读取到 stock=1，判断可以购买。执行 UPDATE stock = 0 。

最终业务效果则是卖出了两件商品，而库存只扣除了 1 个。

可以使用 **写行级锁** 进行避免。如隔离级别为：可重复读，序列化。

## 四、处理并发更新问题

### 乐观锁（Optimistic Locking）

适合读多写少的场景。不依赖数据库底层的锁，而是在数据表里加一个 `version`（版本号）字段或 `updated_at` 时间戳。在每次更新的适合，加上版本号进行精确列更新。

可以提升数据库并发性能（设置隔离级别为读已提交）的同时，保证并发更新冲突的时候进行处理或者 Retry。

### 悲观锁（Pessimistic Locking）

适合写操作多，并发冲突概率极高，或者对数据一致性要求极其严格的场景。依靠数据库底层的锁机制，在读取数据的那一刻就把它锁住。

```sql

BEGIN; 
SELECT balance FROM account WHERE id = 1 FOR UPDATE; -- 此时其他试图修改该行或也执行 FOR UPDATE 的事务会被阻塞 
UPDATE account SET balance = balance - 10 WHERE id = 1; 
COMMIT;

```

对系统并发吞吐量影响大，很容易触发死锁。

### 原子操作

只进行简单的相对数值修改（如增加或减少），不需要在应用层做复杂的逻辑判断。使用 SQL 直接进行操作符运算，避免覆盖写入。利用 UPDATE 自带的行锁。

应用场景较为局限，只适合简单运算。

### 削峰填谷

适用于极端高并发的“热点行”更新（例如秒杀活动中几万人同时抢购同一件商品）。可能短时间内有数万个请求会直接打入数据库中，数据库的行锁会导致严重的性能瓶颈。

引入 Redis 等缓存在缓存层对库存进行预扣除（DECR），依旧要记得避免覆盖写入。或者使用扣减记录异步化，数据库端变成单线程或低并发的顺序排队消费，彻底消除数据库层的“写写冲突”。

## 结

总之对于并发事务情况下，需要根据并发性能与一致性取最优解，根据业务场景量采取适当的隔离级别。没有绝对的最优解。

在特定情况下，可以对事务进行拆分（如涉及库存，第三方调用）等特殊情况，控制事务大小，引入缓存，任务队列等，实现性能与一致性的权衡。