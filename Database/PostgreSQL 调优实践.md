---
title: PostgreSQL 调优实践
date: 2026-04-01
author: Ryan Zeng
tags:
  - postgresql
  - db
categories: []
draft: false
---

## 常用参数

### 内存调优

**shared_buffers**：共享缓冲区，用于设置 PostgreSQL 用于缓存数据的内存大小，通常设置为系统内存的 25% 左右。

PostgreSQL 访问数据时，底层以 page/block 为单位读写。shared_buffers 就是 PG 自己维护的一层页缓存。命中时，读取可以直接从 PG 共享缓存拿；未命中时，再从 OS/磁盘路径取数据，再放入缓存。它的价值不是"把整个库放进内存"，而是让高频访问的热页尽量留在数据库内部缓存中，减少反复 I/O，并让多个会话共享这些页。

启动级参数，需要重启服务。

```sql
-- 查看当前值
SHOW shared_buffers;

SELECT name, setting, unit, context, source
FROM pg_settings
WHERE name = 'shared_buffers';
```

**work_mem**：工作内存，每个查询操作符可用的内存量，直接影响排序、哈希聚合等操作的性能。

复杂查询里可能有多个排序/哈希操作同时存在，且并行查询会把这个资源乘到多个 worker 上，所以总内存消耗可能是 work_mem 的很多倍官方默认 4MB

很容易由于并发和复杂语句导致使用的内存膨胀，谨慎调整。

优化用于：复杂分析，大表 join，排序等

会话级参数

> 不是每个连接，查询，sql 的总内存，是每个节点的内存，如：Sort, Hash join, Hash aggregate 等操作符。

```sql
-- work_mem
-- 每个查询操作符可用的内存量，直接影响排序、哈希聚合等操作的性能。
-- 单个执行节点可用于内部排序、哈希、位图等操作的工作内存上限。
-- 官方定义是：在写入临时磁盘文件前，这些内部操作可使用的基础最大内存；默认值通常是 4MB
SHOW work_mem;

SELECT name, setting, unit, context, source
FROM pg_settings
WHERE name = 'work_mem';
```

### 并发控制

**max_connections**：最大连接数，控制同时连接到数据库的客户端数量，过高可能导致资源争用和性能下降。合理配置相关参数可避免资源争用。默认通常是 100。

```sql
-- max_connections 是 postgresql 实例允许的最大并发连接数。
-- 过多的连接会有更多的内存消耗，进程调度开销，锁竞争等，并发过大时优先考虑使用外部连接池
SHOW max_connections;

SELECT name, setting, unit, context, source
FROM pg_settings
WHERE name IN (
  'max_connections',
  'superuser_reserved_connections',
  'reserved_connections'
);
```

**lock_timeout / statement_timeout**：

lock_timeout 是 postgresql 中的锁等待超时时间设置，单位是毫秒。它定义了一个事务在等待获取锁时的最长时间。如果在这个时间内无法获取到所需的锁，事务将会被中断并抛出一个错误。

statement_timeout 是 postgresql 中的查询执行超时时间设置，单位是毫秒。它定义了一个查询在执行时的最长时间。如果查询执行时间超过这个时间，查询将会被中断并抛出一个错误。

两者都是中断语句并退出，默认都是关闭。

会话级参数，pg 官方推荐使用在特定会话中。用于长时间阻塞请求的保护，DDL 语句，复杂分析（允许长时间查询，不允许长时间锁等待）。

> 如果 statement_timeout 非零，那么把 lock_timeout 设成相同或更大通常没有意义，因为 statement_timeout 总会先触发。
> 所以合理关系通常是：
> lock_timeout < statement_timeout

```sql
SHOW LOCK_TIMEOUT;
SHOW STATEMENT_TIMEOUT;

BEGIN;
SET lock_timeout = '1s';
SET statement_timeout = '30s';
ALTER TABLE "Mall_Goods" ADD COLUMN test_col text;
ROLLBACK;
```

### I/O 调优

**random_page_cost**：随机页成本，random_page_cost 越高，优化器越不愿意走索引路径；越低，越愿意考虑索引路径。

**synchronous_commit**：同步提交设置，控制事务提交时的同步行为，影响数据安全和性能。默认是 on，表示每次提交都等待 WAL 写入磁盘完成以确保数据安全。设置为 off 可以提高性能，但可能在崩溃时丢失最近的事务。

- `on`：完全同步（最高安全性）
- `remote_write`：等待 WAL 远程写入
- `local`：仅本地写入
- `off`：异步提交（最高性能）

**checkpoint**：检查点配置，控制 WAL 写入频率，平衡性能与恢复时间。

- `checkpoint_timeout`：检查点间隔时间，默认 5 分钟。
- `checkpoint_completion_target`：检查点完成目标，控制检查点完成的时间占 checkpoint_timeout 的比例，默认 0.5。
- `max_wal_size`：最大 WAL 大小，控制检查点触发的 WAL 大小，默认 1GB。

> **WAL 是什么？**
> WAL 是 Write-Ahead Log，预写日志。它是 PostgreSQL 用于保证数据一致性和持久性的机制。当数据库执行修改操作（如 INSERT、UPDATE、DELETE）时，首先将这些修改记录写入 WAL 日志文件，然后再将修改应用到数据文件中。这样，即使在系统崩溃的情况下，也可以通过 WAL 日志来恢复数据，确保数据不会丢失。

## 更改参数的方式

PostgreSQL 的参数更改提供四种方式：

1. SQL 命令临时改
2. SQL 命令全局改
3. 启动参数改
4. 配置文件改

**SQL 命令修改**：会话级调参，只在当前会话生效，通常用于调试

```sql
SET work_mem = '128MB';
SET random_page_cost = 1.1;
SET statement_timeout = '30s';

-- 在事务中
BEGIN;
SET LOCAL work_mem = '256MB';
-- 执行测试 SQL
COMMIT;
```

**SQL 命令全局修改**：使用 `ALTER SYSTEM` 修改全局参数，修改后需要重启数据库生效。`ALTER SYSTEM` 会把参数写入 postgresql.auto.conf，该文件和 postgresql.conf 一起读取，而且 postgresql.auto.conf 的设置会覆盖 postgresql.conf。持久化到数据目录。

```sql
ALTER SYSTEM SET work_mem = '128MB';
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET statement_timeout = '30s';
```

**启动参数修改**：postgres 命令支持 `-c name=value` 来设置运行时参数

```yaml
# docker compose
services:
  postgres:
    image: postgres:16
    command:
      - 'postgres'
      - '-c'
      - 'shared_buffers=1GB'
      - '-c'
      - 'work_mem=32MB'
      - '-c'
      - 'max_connections=200'
      - '-c'
      - 'checkpoint_timeout=15min'
```

**配置文件修改**：使用 `postgresql.conf` 持久化配置信息进行修改，这是官方最推荐的方式。

```yaml
# docker compose
services:
  postgres:
    image: postgres:16
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
      - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
    command:
      - 'postgres'
      - '-c'
      - 'config_file=/etc/postgresql/postgresql.conf'
```

## 如何进行 Explain

**EXPLAIN**：优化器的估算，不算真正执行

- `cost`：优化器的成本估算，不是实际耗时
- `rows`：该节点预计输出的行数
- `width`：预计每行平均字节数

```sql
--"Seq Scan on ""Mall_Goods""  (cost=0.00..631.30 rows=12021 width=878)"
--"  Filter: (""Price"" > '100'::numeric)"
EXPLAIN SELECT * FROM "Mall_Goods" WHERE "Price" > 100;
```

**EXPLAIN ANALYZE**：真正执行

```sql
-- "Seq Scan on ""Mall_Goods""  (cost=0.00..631.30 rows=12021 width=878) (actual time=0.019..2.686 rows=11719 loops=1)"
-- "  Filter: (""Price"" > '100'::numeric)"
--   Rows Removed by Filter: 3280
-- Planning Time: 0.071 ms
-- Execution Time: 2.971 ms
EXPLAIN ANALYZE SELECT * FROM "Mall_Goods" WHERE "Price" > 100;
```

**EXPLAIN (ANALYZE, BUFFERS)**：最常用形式，还会显示缓存使用情况

```sql
-- "Seq Scan on ""Mall_Goods""  (cost=0.00..631.30 rows=12021 width=878) (actual time=0.010..2.141 rows=11719 loops=1)"
-- "  Filter: (""Price"" > '100'::numeric)"
--   Rows Removed by Filter: 3280
--   Buffers: shared hit=439
-- Planning Time: 0.061 ms
-- Execution Time: 2.463 ms
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM "Mall_Goods" WHERE "Price" > 100;
```

**综合示例**：多表 JOIN 查询

```sql
-- Hash Left Join  (cost=258.41..944.43 rows=6195 width=1166) (actual time=1.238..4.889 rows=6081 loops=1)
-- "  Hash Cond: (mg.""MainPictureId"" = sa.""Id"")"
--   Buffers: shared hit=610
-- "  ->  Seq Scan on ""Mall_Goods"" mg  (cost=0.00..669.76 rows=6195 width=878) (actual time=0.007..2.476 rows=6081 loops=1)"
-- "        Filter: ((""Price"" > '100'::numeric) AND ((""WaresName"")::text ~~ '%茶%'::text))"
--         Rows Removed by Filter: 8918
--         Buffers: shared hit=439
--   ->  Hash  (cost=209.85..209.85 rows=3885 width=288) (actual time=1.214..1.215 rows=3944 loops=1)
--         Buckets: 4096  Batches: 1  Memory Usage: 719kB
--         Buffers: shared hit=171
-- "        ->  Seq Scan on ""Sys_Attachments"" sa  (cost=0.00..209.85 rows=3885 width=288) (actual time=0.005..0.377 rows=3944 loops=1)"
--               Buffers: shared hit=171
-- Planning:
--   Buffers: shared hit=238
-- Planning Time: 0.560 ms
-- Execution Time: 5.061 ms
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM "Mall_Goods" AS mg
LEFT JOIN "Sys_Attachments" AS sa
ON mg."MainPictureId" = sa."Id"
WHERE mg."Price" > 100 AND mg."WaresName" LIKE '%茶%';
```

## 注意

- **渐进式调优**：建立性能基线，逐步调整参数，每次只调整一个参数，观察性能变化。
- **防止过度配置**：work_mem 和 max_connections 等参数过高可能导致内存不足，导致系统崩溃。合理评估系统资源和负载需求，避免过度配置。
- **监控和日志**：使用 pg_stat_io 和 pg_stat_database、pg_stat_bgwriter，针对性地进行优化。

## 相关资料

- [**Postgresql Conf:**]('https://postgresqlco.nf/') `postgresql.conf` 参数文档查阅，参数测试网站。