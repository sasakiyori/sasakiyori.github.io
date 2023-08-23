---
layout: article
title: Postgres使用PgAudit获取审计日志
tags: postgres PostgreSQL PgAudit
key: 2023-08-23-postgres-audit
---

## 背景

数据库的使用过程中，执行SQL的历史经常是需要被记录且审计的，例如在政府、企业等使用场景。

## 实现

本次使用[PgAudit](https://github.com/pgaudit/pgaudit/tree/master)以插件的形式接入Postgres，实现审计日志的打印。  
PgAudit审计信息与Postgres服务日志是放在一起的，筛选起来有难度，且[官方不会做相关的支持](https://github.com/pgaudit/pgaudit/issues/63)。  
因此我额外使用[PgAuditLogToFile](https://github.com/fmbiete/pgauditlogtofile)来作为PgAudit的辅助，将审计日志放到独立文件中，方便进行采集和分析。

### 插件编译安装

在postgres所在容器上，需要编译安装上述两个插件的动态库:

```shell
# PgAudit编译安装

# 因为我使用postgres v12，因此使用PgAudit对应的v1.4.X版本或者REL_12_STABLE分支
git clone https://github.com/pgaudit/pgaudit.git
git checkout v1.4

# PG_CONFIG指定pg_config的所在路径
# postgresql默认为/usr/local/bin/pg_config
USE_PGXS=1 PG_CONFIG=/usr/local/bin/pg_config make install


# PgAuditLogToFile安装
git clone https://github.com/fmbiete/pgauditlogtofile.git
USE_PGXS=1 PG_CONFIG=/usr/local/bin/pg_config make install


# 上述make install执行后，相应的动态库被安装到pg的动态库文件夹中
# postgresql默认为/usr/local/lib/postgresql
```

### 插件预加载

```ini
# postgresql.conf

# 修改配置文件后必须重启服务，发送SIGHUP信号是无效的
shared_preload_libraries = 'pgaudit,pgauditlogtofile'
```

### 创建插件

登录postgres，创建对应的插件：

```sql
CREATE EXTENSION pgaudit;
CREATE EXTENSION pgauditlogtofile;
```

### 设置PgAudit配置

全局设置PgAudit的配置后，重启服务。这一步也可以提前在`postgresql.conf`中配置好：

```sql
-- 官方提示这一步要在CREATE EXTENSION之后: 
-- In addition, `CREATE EXTENSION pgaudit` must be called before `pgaudit.log` is set to ensure proper pgaudit functionality.
-- 可选项有: READ/WRITE/FUNCTION/ROLE/DDL/MISC/MISC_SET/ALL
ALTER SYSTEM SET pgaudit.log='all';
-- 设置为on的话不会打印到日志文件上
ALTER SYSTEM SET pgaudit.log_client=off;
ALTER SYSTEM SET pgaudit.log_directory ='/var/lib/postgresql/data/log';
ALTER SYSTEM SET pgaudit.log_level=info;

SELECT pg_reload_conf();
```

## 审计日志

得到的审计结果格式为csv，例如：

```csv
2023-08-23 01:55:28.175 UTC,postgres,postgres,279,[local],64e56702.117,1,SELECT,2023-08-23 01:55:14 UTC,3/80,0,00000,SESSION,1,1,READ,SELECT,,,select pg_sleep(10);,<not logged>,,,,,,,,,psql
```
