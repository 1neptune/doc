# Vastbase G100 日常运维手册

> 适用版本：Vastbase G100 V3.0
> 集群模式：主节点 + 备节点 + 仲裁节点的 HAS 集群
> 实际数据目录：`/data/vastdata`
> 实际备份目录：`/data/backup`
> pb_probackup 实例名：`BACKUP`
> 备份日志目录：`/home/vastbase/vbbackup`
> 命令行工具：`vsql`、`vb_ctl`、`vb_guc`、`vb_probackup`、`vb_dump`、`vb_restore`、`has_ctl`
> 运维账号：数据库操作使用操作系统用户 `vastbase`，集群管理部分涉及 `root`

---

## 1. 连接数据库

以下命令均以操作系统用户 `vastbase` 在数据库主节点执行。

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 1.1 | 连接数据库 | `vsql` |
| 1.2 | 连接指定数据库（默认 vastbase 库） | `vsql -d vastbase -p 5432` |
| 1.3 | 远程连接数据库 | `vsql -d postgres -h 10.10.0.11 -U jack -p 5432 -W Test@123` |
| 1.4 | 退出 vsql | `\q` |
| 1.5 | 查看当前数据库版本 | `SELECT version();` |
| 1.6 | 查看当前连接的用户与数据库 | `\conninfo` |

---

## 2. 集群与实例状态查询

### 2.1 实例状态（单机层面）

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 2.1.1 | 查看数据库实例是否运行 | `vb_ctl status [-D /data/vastdata]` |
| 2.1.2 | 查看主备节点角色（主库返回 primary，备库返回 standby，仲裁节点返回 standby） | `vsql -t -A -c "select case when pg_is_in_recovery()='f' then 'primary' else 'standby' end;"` |

### 2.2 集群状态（HAS / DCS）

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 2.2.1 | 查看集群状态（各节点角色、状态、延迟） | `has_ctl query -C -v -i` |
| 2.2.2 | 验证 DCS(etcd) 成员列表 | `cd /vastbase/dcs && ./etcdctl member list` |
| 2.2.3 | 验证 DCS(etcd) 集群健康状态 | `cd /vastbase/dcs && ./etcdctl cluster-health` |
| 2.2.4 | 查看 HAS 集群成员列表 | `cd /vastbase/has && ./hasctl -c vastbase.yml list` |

---

## 3. 数据库 / 集群启停

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 3.1 | 启动数据库实例（单机） | `vb_ctl start` |
| 3.2 | 停止数据库实例（单机） | `vb_ctl stop` |
| 3.3 | 设置启动超时时间（默认 60s） | `vb_ctl start --time-out=60` |
| 3.4 | 启动整个集群（在任一节点执行） | `has_ctl start` |
| 3.5 | 停止整个集群（在任一节点执行） | `has_ctl stop` |
| 3.6 | 停止指定节点（故障切换演练用） | `has_ctl stop -n NODEID` |
| 3.7 | 重启数据库实例 | `vb_ctl restart` |

---

## 4. 主备切换与集群维护

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 4.1 | 计划内主备切换（指定节点升为主） | `has_ctl switchover -n NODEID -D /data/vastdata` |
| 4.2 | 切换后查看集群状态确认 | `has_ctl query -C -v -i` |
| 4.3 | 修改集群配置（如 max_connections） | `/vastbase/has/hasctl -c /vastbase/has/vastbase.yml edit-config` |
| 4.4 | 修改配置后重启 has 使配置生效（root 用户） | `systemctl restart has` |
| 4.5 | 修改实例参数并 reload 生效 | `vb_guc reload -D /data/vastdata -c "max_connections=200"` |
| 4.6 | 修改实例参数并写入配置文件（需重启） | `vb_guc set -D /data/vastdata -c "max_process_memory=10GB"` |
| 4.7 | 卸载清理集群（删除数据） | `gs_uninstall --delete-data` |
| 4.8 | 本地卸载清理集群 | `gs_uninstall --delete-data -L` |

> 注意：`switchover` 为维护类操作，应在数据库状态正常、业务结束且主备无追赶后进行。

---

## 5. 备份（vb_probackup）

实际配置：数据目录 `/data/vastdata`、备份目录 `/data/backup`、实例名 `BACKUP`、保留策略 `retention-redundancy=3`、`retention-window=30`、压缩 `zlib level 5`。

### 5.1 初始化与配置

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 5.1.1 | 初始化备份目录（首次使用） | `vb_probackup init -B /data/backup` |
| 5.1.2 | 添加备份实例（首次使用） | `vb_probackup add-instance -B /data/backup -D /data/vastdata --instance BACKUP` |
| 5.1.3 | 设置 / 查看备份留存与压缩配置 | `vb_probackup set-config -B /data/backup --instance BACKUP --retention-window=30 --retention-redundancy=3 --compress-algorithm zlib --compress-level 5 --log-level-file=info --log-directory=/home/vastbase/vbbackup --log-filename=full_backup-%Y%m%d.log` |
| 5.1.4 | 查看备份配置内容 | `vb_probackup show-config -B /data/backup --instance BACKUP` |

### 5.2 执行备份

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 5.2.1 | 全量备份（-j 为并行线程数） | `vb_probackup backup -B /data/backup --instance BACKUP --stream -b full -j 4` |
| 5.2.2 | 增量备份（PTRACK，需先有全量备份） | `vb_probackup backup -B /data/backup --instance BACKUP --stream -b PTRACK -j 4` |
| 5.2.3 | 备份时指定数据库用户与密码 | `vb_probackup backup -B /data/backup --instance BACKUP --stream -b full -j 4 -U backup -W Back#1234` |
| 5.2.4 | 查看全部备份集信息 | `vb_probackup show -B /data/backup --instance BACKUP` |
| 5.2.5 | 查看指定备份集详情 | `vb_probackup show -B /data/backup --instance BACKUP -i backup_id` |
| 5.2.6 | 校验备份完整性 | `vb_probackup validate -B /data/backup --instance BACKUP` |
| 5.2.7 | 合并增量备份到父全量备份 | `vb_probackup merge -B /data/backup --instance BACKUP -i backup_id` |
| 5.2.8 | 删除过期备份与 WAL（wal 保留 4 份） | `vb_probackup delete -B /data/backup --instance BACKUP --delete-expired --delete-wal --wal-depth 4` |
| 5.2.9 | 删除指定状态为 ERROR 的失败备份集 | `vb_probackup delete -B /data/backup --instance BACKUP --status=ERROR` |
| 5.2.10 | 删除指定备份集 | `vb_probackup delete -B /data/backup --instance BACKUP -i backup_id` |

### 5.3 定时备份脚本

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 5.3.1 | 执行主备自动备份脚本（主库周日全备、其余增量，备库清理归档与 CBM） | `bash /home/vastbase/vbbackup/vb_backup.sh` |
| 5.3.2 | 添加 crontab 定时任务（例：每日 02:00 执行） | `crontab -e` → `0 2 * * * bash /home/vastbase/vbbackup/vb_backup.sh >> /home/vastbase/vbbackup/backup.log 2>&1` |
| 5.3.3 | 查看定时任务 | `crontab -l` |

> 前置条件：需在 `postgresql.conf` 中开启 `enable_cbm_tracking = on` 并重启数据库，增量备份前必须先做一次全量备份。

---

## 6. 恢复（vb_probackup）

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 6.1 | 按备份集恢复到指定数据目录（先停止数据库，目录可自动创建） | `vb_probackup restore -B /data/backup --instance BACKUP -D /data/backup1 -i backup_id` |
| 6.2 | 恢复到最新备份（不指定 -i） | `vb_probackup restore -B /data/backup --instance BACKUP -D /data/backup1` |
| 6.3 | 恢复到指定时间点（PITR） | `vb_probackup restore -B /data/backup --instance BACKUP -D /data/backup1 --recovery-target-time='2026-08-11 02:00:00'` |
| 6.4 | 恢复到指定 LSN | `vb_probackup restore -B /data/backup --instance BACKUP -D /data/backup1 --recovery-target-lsn=0/66000028` |
| 6.5 | 恢复到指定事务 XID | `vb_probackup restore -B /data/backup --instance BACKUP -D /data/backup1 --recovery-target-xid=1000` |
| 6.6 | 恢复时强制覆盖非空目录 | `vb_probackup restore -B /data/backup --instance BACKUP -D /data/backup1 --force-overwrite` |
| 6.7 | 恢复完成后启动数据库 | `vb_ctl start -D /data/backup1` |
| 6.8 | 恢复后连接验证数据 | `vsql -d vastbase -p 5432 -r` |

> 使用 restore 前应先停止数据库进程；备份与恢复的服务器主版本号必须一致。

---

## 7. 逻辑备份与恢复

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 7.1 | 指定用户导出整个数据库 | `vb_dump dbname -p 5432 -f /home/vastbase/out.sql -U user_name -W password` |
| 7.2 | 只导出指定 schema | `vb_dump dbname -p 5432 -n schema_name -f /home/vastbase/out.sql` |
| 7.3 | 只导出指定表 | `vb_dump dbname -p 5432 -t table_name -f /home/vastbase/out.sql` |
| 7.4 | 导出整个集群所有数据库 | `vb_dumpall -f /home/vastbase/all.sql` |
| 7.5 | 逻辑恢复（导入转储文件） | `vb_restore -d dbname -p 5432 /home/vastbase/out.sql` |
| 7.6 | 使用 vsql 执行 SQL 脚本恢复 | `vsql -d dbname -p 5432 -f /home/vastbase/out.sql` |

---

## 8. 例行日常检查

### 8.1 基本运维检查

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 8.1.1 | 检查数据库实例状态 | `vb_ctl status` |
| 8.1.2 | 查询当前数据库连接的会话数 | `SELECT count(*) FROM pg_stat_activity;` |
| 8.1.3 | 查看最大连接数 | `SHOW max_connections;` |
| 8.1.4 | 查看空闲连接（state=idle） | `SELECT * FROM pg_stat_activity WHERE state='idle' ORDER BY state_change;` |
| 8.1.5 | 终止占用的空闲连接（pid 为连接进程号） | `SELECT pg_terminate_backend(pid);` |
| 8.1.6 | 查看会话时间（backend/xact/query 启动时间） | `SELECT backend_start,xact_start,query_start,state_change FROM pg_stat_activity;` |
| 8.1.7 | 查看占用内存最多的会话 | `SELECT * FROM pv_session_memory_detail() ORDER BY usedsize desc limit 10;` |

### 8.2 锁与事务检查

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 8.2.1 | 查询数据库中的锁信息 | `SELECT * FROM pg_locks;` |
| 8.2.2 | 查询等待锁的线程状态 | `SELECT * FROM pg_thread_wait_status WHERE wait_status = 'acquire lock';` |
| 8.2.3 | 查看当前数据库进程 | `ps ux` |
| 8.2.4 | 结束指定系统进程 | `kill -9 pid` |

### 8.3 容量与统计

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 8.3.1 | 查看表占用的空间 | `SELECT pg_table_size('table_name');` |
| 8.3.2 | 查看数据库占用的空间 | `SELECT pg_database_size('database_name');` |
| 8.3.3 | 查看表结构 | `\d+ table_name` |
| 8.3.4 | 查看索引结构 | `\d+ index_name` |
| 8.3.5 | 查看表统计信息 | `SELECT * FROM pg_statistic;` |
| 8.3.6 | 查看分区表信息 | `SELECT * FROM pg_partition;` |
| 8.3.7 | 查看约束信息 | `SELECT * FROM pg_constraint;` |

### 8.4 时间一致性检查（月度）

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 8.4.1 | 编辑节点清单文件 | `vim /tmp/mpphosts` |
| 8.4.2 | 采集各节点时间到日志文件 | `for ihost in 'cat /tmp/mpphosts'; do ssh -n -q $ihost "hostname;date"; done > /tmp/sys_ctl-os1.log` |
| 8.4.3 | 查看各节点时间（差异应小于 30 秒） | `cat /tmp/sys_ctl-os1.log` |

---

## 9. 例行维护（表 / 索引）

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 9.1 | 对表执行 VACUUM 回收空间并更新统计信息 | `VACUUM table_name;` |
| 9.2 | 对表分区执行 VACUUM | `VACUUM table_name PARTITION(p1);` |
| 9.3 | 对表执行 VACUUM FULL（回收空间合并小文件，需排他锁） | `VACUUM FULL table_name;` |
| 9.4 | 收集表统计信息 | `ANALYZE table_name;` |
| 9.5 | 收集统计信息并输出详情 | `ANALYZE VERBOSE table_name;` |
| 9.6 | 同时执行 VACUUM 与 ANALYZE | `VACUUM ANALYZE table_name;` |
| 9.7 | 重建索引（REINDEX，会加排他锁） | `REINDEX TABLE table_name;` |
| 9.8 | 重建列存表内部索引 | `REINDEX INTERNAL TABLE cgin_create_test;` |
| 9.9 | 重建索引方式一：先删后建 | `DROP INDEX index_name;` → `CREATE INDEX index_name ON table_name (col);` |
| 9.10 | 查看 SQL 执行计划 | `EXPLAIN your_sql;` |

---

## 10. 日志检查与清理

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 10.1 | 查看操作系统日志（关注 kernel/error/fatal） | `vim /var/log/messages` |
| 10.2 | 查看备份日志目录 | `ls /home/vastbase/vbbackup/` |
| 10.3 | 查看最近的全量备份日志 | `ls -lt /home/vastbase/vbbackup/full_backup-*.log` |
| 10.4 | 按月清理历史备份日志 | `find /home/vastbase/vbbackup -mtime +30 -name "*.log" -exec rm -f {} \;` |

---

## 11. 慢 SQL 与性能诊断

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 11.1 | 查看全量 SQL 执行信息 | `SELECT * FROM dbe_perf.get_global_full_sql_by_timestamp('start_ts','end_ts');` |
| 11.2 | 查看慢 SQL 执行信息 | `SELECT * FROM dbe_perf.get_global_slow_sql_by_timestamp('start_ts','end_ts');` |
| 11.3 | 查看 SQL 语句执行历史 | `SELECT * FROM statement_history;` |
| 11.4 | 开启 SQL 跟踪级别（建议 L0,L0） | `vb_guc set -D /data/vastdata -c "track_stmt_stat_level=L0,L0"` |

### WDR 诊断报告

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 11.5 | 新建报告文件 | `touch /home/vastbase/wdrTestNode.html` |
| 11.6 | 查看最近快照 | `SELECT * FROM snapshot.snapshot ORDER BY start_ts DESC LIMIT 10;` |
| 11.7 | 设置报告输出格式与文件 | `\a \t \o /home/vastbase/wdrTestNode.html` |
| 11.8 | 生成集群级 WDR 报告 | `SELECT generate_wdr_report(1, 2, 'all', 'cluster', null);` |
| 11.9 | 生成节点级 WDR 报告 | `SELECT generate_wdr_report(1, 2, 'all', 'node', pgxc_node_str()::cstring);` |
| 11.10 | 关闭输出选项 | `\o \a \t` |

---

## 12. 安全与审计

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 12.1 | 修改用户密码 | `ALTER ROLE user_name IDENTIFIED BY 'Newpwd123' REPLACE 'Oldpwd123';` |
| 12.2 | 创建用户 | `CREATE USER jack PASSWORD 'Test@123';` |
| 12.3 | 查看审计结果 | `SELECT * FROM pg_query_audit();` |
| 12.4 | 查看 pg_hba 认证配置 | `cat $PGDATA/pg_hba.conf` |
| 12.5 | 添加客户端接入认证规则 | `vb_guc reload -D /data/vastdata -h "host all jack 10.10.0.30/32 sha256"` |
| 12.6 | 查看远程监听配置 | `cat $PGDATA/postgresql.conf | grep listen_addresses` |

---

## 13. 闪回恢复（误操作恢复）

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 13.1 | 设置 undo 旧版本保留时间（600 秒） | `ALTER SYSTEM SET undo_retention_time TO 600;` |
| 13.2 | 查看 undo_retention_time 是否生效 | `SHOW undo_retention_time;` |
| 13.3 | 闪回表到指定时间戳 | `TIMECAPSULE TABLE t1 TO TIMESTAMP to_timestamp('2026-08-11 10:13:22','YYYY-MM-DD HH24:MI:SS');` |
| 13.4 | 闪回表到指定 CSN | `TIMECAPSULE TABLE t1 TO CSN 9617;` |

> 闪回表需使用 USTORE 存储引擎，且需配置 `enable_default_ustore_table=on` 或建表时指定 `WITH(STORAGE_TYPE=USTORE)`。

---

## 14. 常用运维捷径 / 速查

| 编号 | 运维作用 | 运维命令 |
|------|----------|----------|
| 14.1 | 查看当前集群主备角色 | `vsql -t -A -c "select pg_is_in_recovery();"` |
| 14.2 | 一键查看集群状态清单 | `has_ctl query -C -v -i` |
| 14.3 | 一键查看备份清单 | `vb_probackup show -B /data/backup --instance BACKUP` |
| 14.4 | 一键校验备份 | `vb_probackup validate -B /data/backup --instance BACKUP` |

---

## 附：备份脚本 vb_backup.sh 说明

脚本位于 `/home/vastbase/vbbackup/vb_backup.sh`，功能逻辑如下：

1. 先判断本节点角色（primary / standby）。
2. **主节点**：
   - 周日或上次未成功全备时执行**全量备份**，随后删除过期备份与 WAL、清理 CBM。
   - 其余日期执行 **PTRACK 增量备份**，随后删除过期备份与 WAL、清理 CBM。
3. **备节点**：仅执行删除过期备份与 WAL、清理备库 CBM，不做备份（避免重复）。

关键参数：
- 备份目录：`/data/backup`
- 数据目录：`/data/vastdata`
- 实例名：`BACKUP`
- 并行线程数：`job_num=4`
- 保留策略：冗余 3 份、保留 30 天
- WAL 保留深度：4

> 建议通过 crontab 每日定时执行，并定期检查 `/home/vastbase/vbbackup` 下的备份日志确认备份结果。