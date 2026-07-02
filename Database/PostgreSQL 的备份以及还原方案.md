## 使用 pg_dump 配合 cron_job 进行定时备份

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
- 基础备份 base backup
- 从基础备份开始之后连续完整的 WAL 归档
### WAL = Write-Ahead Log，预写日志

PostgreSQL 修改数据时，不是只改数据文件，而是先把变更记录写进 WAL。

大致流程：

```
用户执行 SQL  ↓PostgreSQL 生成 WAL 记录  ↓事务提交  ↓后台进程慢慢把脏页刷入数据文件
```

所以 WAL 里记录了数据库发生过的变更。


## 使用第三方服务进行备份

### Databasus

Databasus 是一款免费、开源且可自行托管的 PostgreSQL 备份工具。它支持使用不同的存储介质（S3、Google Drive、FTP 等）进行备份，并通过 Slack、Discord、Telegram 等渠道发送进度通知。Databasus 专注于实现低 RPO/RTO 的时间点恢复。

基于 pg_dump 实现，拓展功能，在此基础上拓展定时任务，web 面板，通知提醒，云存储集成等。

> 项目地址： https://github.com/databasus/databasus

Docker Compose 文件

```yaml
services:
  # PostgreSQL 数据库服务
  postgres:
    image: postgres:17.4
    container_name: postgres_db
    environment:
      POSTGRES_USER: ${POSTGRES_USER} # 数据库用户名
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD} # 数据库密码
      POSTGRES_DB: ${POSTGRES_DB} # 默认数据库名
    ports:
      - '8432:5432' # 主机端口:容器端口
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data # 数据持久化卷
      - postgres_wal_archive:/var/lib/postgresql/wal_archive # WAL归档卷
    command:
      - postgres
      - -c
      - wal_level=replica
      - -c
      - archive_mode=on
      - -c
      - 'archive_command=if [ ! -f /var/lib/postgresql/wal_archive/%f ]; then cp %p /var/lib/postgresql/wal_archive/%f; else true; fi'
      - -c
      - archive_timeout=60s
      - -c
      - max_wal_senders=10
      - -c
      - wal_keep_size=1GB
    restart: unless-stopped
    networks:
      - postgres-dev-network
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}']
      interval: 10s
      timeout: 5s
      retries: 5

  # Databasus 数据库备份
  databasus:
    container_name: databasus
    image: databasus/databasus:v3.47.1
    ports:
      - "4005:4005"
    volumes:
      - databasus_data:/databasus-data
    restart: unless-stopped
    depends_on:
      - postgres
    networks:
      - postgres-dev-network
    healthcheck:
      test: ["CMD", "databasus", "healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s

networks:
  postgres-dev-network:
    driver: bridge

volumes:
  postgres_dev_data:
  postgres_wal_archive:
  databasus_data:

```