## 使用 pg_dump 配合 cron_job 进行定时备份

### pg_dump是什么？

`pg_dump`是 PostgreSQL 的原生自带实用程序，用于创建逻辑备份, 是数据库导出的标准工具。

传统的 pg_dump 脚本

```bash
#!/bin/bash
# Backup script for pg_dump
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"
DB_NAME="mydb"

# Create backup
pg_dump -Fc -h localhost -U postgres $DB_NAME > $BACKUP_DIR/$DB_NAME_$DATE.dump

# Compress (if not using custom format)
# gzip $BACKUP_DIR/$DB_NAME_$DATE.sql

# Encrypt
gpg --encrypt --recipient backup@company.com $BACKUP_DIR/$DB_NAME_$DATE.dump

# Upload to S3
aws s3 cp $BACKUP_DIR/$DB_NAME_$DATE.dump.gpg s3://my-bucket/backups/

# Cleanup old backups (keep last 7 days)
find $BACKUP_DIR -name "*.dump*" -mtime +7 -delete

# Send notification on failure
if [ $? -ne 0 ]; then
  curl -X POST https://hooks.slack.com/... -d '{"text":"Backup failed!"}'
fi
```

### 配置 CronJob 定时执行备份

/opt/scripts/pg_backup.sh 内容：

````bash
#!/usr/bin/env bash

set -e

# PostgreSQL 连接信息
PG_HOST="127.0.0.1"
PG_PORT="8432"
PG_USER="backup_user"
PG_PASSWORD="your_password"
PG_DATABASE="rcerp_db"

# 备份配置
BACKUP_DIR="/var/backups/postgresql"
RETENTION_DAYS=14
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="${BACKUP_DIR}/${PG_DATABASE}_${TIMESTAMP}.dump"

mkdir -p "$BACKUP_DIR"

echo "开始备份数据库：${PG_DATABASE}"

PGPASSWORD="$PG_PASSWORD" pg_dump \
  -h "$PG_HOST" \
  -p "$PG_PORT" \
  -U "$PG_USER" \
  -d "$PG_DATABASE" \
  -F c \
  -Z 6 \
  -f "$BACKUP_FILE"

echo "备份完成：${BACKUP_FILE}"

find "$BACKUP_DIR" \
  -type f \
  -name "${PG_DATABASE}_*.dump" \
  -mtime "+${RETENTION_DAYS}" \
  -delete

echo "已清理超过 ${RETENTION_DAYS} 天的备份"

````

保存为：

```bash
# 保存脚本文件
sudo vim /opt/scripts/pg_backup.sh

# 添加执行权限
sudo chmod +x /opt/scripts/pg_backup.sh

# 测试
sudo /opt/scripts/pg_backup.sh

# 配置 Cron 每天凌晨两点执行
# 0 2 * * * /opt/scripts/pg_backup.sh >> /var/log/pg_backup.log 2>&1
sudo crontab -e

# 恢复备份
PGPASSWORD="your_password" pg_restore \
  -h 127.0.0.1 \
  -p 8432 \
  -U postgres \
  -d rcerp_db_restore \
  --clean \
  --if-exists \
  /var/backups/postgresql/rcerp_db_20260702_020000.dump
```
### pg_dump 优势

- **便携式备份**：创建 SQL 或自定义格式的转储，可以恢复到不同的 PostgreSQL 版本。
- **选择性备份**：可以导出特定表、模式或整个数据库。
- **一致性快照**：使用 PostgreSQL 的 MVCC 创建一致性备份，而不会阻塞写入。
- **广泛支持**：适用于所有 PostgreSQL 安装，文档齐全，久经考验。
- **灵活的输出格式**：纯 SQL、自定义、目录或 tar 格式。

### pg_dump 的限制

虽然`pg_dump`功能强大，但在生产环境中使用通常需要额外的脚本编写：

- **没有内置调度功能**：需要 cron 作业或外部调度程序。
- **仅限本地存储**：输出到本地文件系统；云上传需要额外的脚本。
- **无加密**：备份文件默认情况下未加密；需要通过 gpg 或类似工具进行管道传输。
- **无通知**：如果没有自定义脚本，就无法在备份成功或失败时发出警报。
- **没有保留管理**：旧备份必须手动或通过脚本清理。
- **仅命令行**：没有用于监控或管理的图形界面。

## 使用 PITR 进行完整备份

### PITR = Point-In-Time Recovery，时间点恢复。

流程类似于 

```
02:00 做了一次基础备份  
10:31 用户误删表  
10:35 发现事故  
  
普通备份：只能恢复到 02:00  
PITR：可以恢复到 10:30:59
```

依赖于：
- 基础备份 basebackup
- 从基础备份开始之后连续完整的 WAL 归档
### WAL = Write-Ahead Log，预写日志

PostgreSQL 修改数据时，不是只改数据文件，而是先把变更记录写进 WAL。

大致流程：

```
用户执行 SQL  ↓PostgreSQL 生成 WAL 记录  ↓事务提交  ↓后台进程慢慢把脏页刷入数据文件
```

所以 WAL 里记录了数据库发生过的变更。

> 注意，它不是从当前数据库往回“撤销”。 从过去的基础备份开始，向前重放 WAL,本质是前置回滚。

> 与 **mysql** 的 binlog 的区别： PostgreSQL 在真正修改数据文件之前，先写入 WAL。WAL 描述的是数据库内部发生的物理或生理层变更，它并不是 SQL 执行历史。保存执行该 SQL 后产生的底层存储变更。 由 pg 的 mvcc 架构以及一致性考虑而设计的。
### 基础备份

基础备份是 使用  **pg_basebackup** 生成的物理副本。 例：

```bash
pg_basebackup \  -h 127.0.0.1 \  -U replication_user \  -D /backup/base_20260701 \  -Fp \  -Xs \  -P
```

> 与 pg_dump 的区别： pg_dump 属于是逻辑备份，备份的是表结构，数据等。 pg_basebackup 属于是物理备份。

## 使用第三方服务进行备份

### Databasus

> Databasus 与其他备份工具的比较：https://databasus.com/pgdump-alternative

Databasus 是一款免费、开源且可自行托管的 PostgreSQL 备份工具。它支持使用不同的存储介质（S3、Google Drive、FTP 等）进行备份，并通过 Slack、Discord、Telegram 等渠道发送进度通知。Databasus 专注于实现低 RPO/RTO 的时间点恢复。

基于 pg_dump 实现，拓展功能，在此基础上拓展定时任务，web 面板，通知提醒，云存储集成等。

> 项目地址： https://github.com/databasus/databasus

Docker Compose 文件

```yaml

services:
  # Databasus 数据库备份
  databasus:
    container_name: databasus
    image: databasus/databasus:v3.47.1
    ports:
      - "4005:4005"
    volumes:
      - databasus_data:/databasus-data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "databasus", "healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s

volumes:
  databasus_data:

```

### pg_dump 逻辑备份

`add database -> Logical -> 创建只读用户 -> 配置通知渠道/备份周期`
### PITR 物理全量备份

需要在 pg 上配置：

```
PostgreSQL 17+  
summarize_wal = on  
wal_level = replica  
创建 LOGIN + REPLICATION 用户  
pg_hba.conf 允许 replication 连接  
确保 Databasus 能访问 PostgreSQL  
在 UI 选择 FULL_INCREMENTAL_AND_WAL_STREAM
```

sql：
```sql
-- PITR 需要: wal_level：replica , summarize_wal：on  
  
-- wal_level 决定 PostgreSQL 在 WAL 中记录多少信息  
-- 值：minimal, replica（默认）, logical  
SHOW wal_level;  
  
-- summarize_wal 启动 PostgreSQL 的 WAL summarizer 后台进程。扫描 WAL，生成摘要文件，记录某个 WAL 范围内哪些数据块发生过变化。  
-- 服务于：pg_basebackup --incremental=上一份备份的backup_manifest  
-- 执行原生增量备份，必须启用 WAL summarization；summarize_wal 默认是 off （原生增量备份： pg 17 的新功能）  
-- 启用 ALTER SYSTEM SET summarize_wal = 'on'; SELECT pg_reload_conf();SHOW summarize_wal;  
  
-- archive_mode archive_mode 控制 PostgreSQL 是否启用 WAL 归档机制。  
-- 值：off（默认）, on, always(备用库)  
SHOW archive_mode;  
  
  
-- 配置  
ALTER SYSTEM SET wal_level = 'replica';  
ALTER SYSTEM SET summarize_wal = 'on';  
ALTER SYSTEM SET wal_summary_keep_time = '30d';  
ALTER SYSTEM SET archive_mode = 'on';  
ALTER SYSTEM SET archive_timeout = '60s';  
ALTER SYSTEM SET max_wal_senders = '10';  
SELECT pg_reload_conf();
```

配置 PostgreSQL 实例允许建立replication 连接。

```sql
-- 查询是否有权限
SELECT  
rolname,  
rolcanlogin,  
rolreplication  
FROM pg_roles  
WHERE rolname = 'databasus_backup';

-- 无权限时执行
ALTER ROLE databasus_backup WITH LOGIN REPLICATION;

-- 查询 hba 配置文件路径
SHOW hba_file;

```

插入相关白名单配置(ip地址可以为 docker 网段，或者 compose 内容器 ip，也可以是公网 ipv4)

```conf
host    replication    databasus_backup    172.20.0.0/16    scram-sha-256
```

在 Web DashBoard 进行配置
`add Database -> Physical`