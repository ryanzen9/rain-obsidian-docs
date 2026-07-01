## 使用 pg_dump 配合 cron_job 进行定时备份

## 使用 PITR 进行完整备份

**PITR = Point-In-Time Recovery，时间点恢复。**

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

