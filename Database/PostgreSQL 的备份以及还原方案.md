## 使用 pg_dump 配合 cron_job 进行定时备份

### pg_dump是什么？

`pg_dump`是 PostgreSQL 的原生实用程序，用于创建逻辑备份。它从 PostgreSQL 诞生之初就已存在，是数据库导出的标准工具。

### 定时任务

pg_backup.sh 内容：

```bash

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