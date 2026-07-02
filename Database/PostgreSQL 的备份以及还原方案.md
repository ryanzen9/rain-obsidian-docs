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

基于 pg_dump 实现，拓展功能，在此基础上拓展定时任务，web 面板，通知提醒，存储

> 项目地址： https://github.com/databasus/databasus